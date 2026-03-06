# I. 功能模块拆分

## 1. 模块总览与 MVP 标注

| 编号 | 模块名称 | MVP | 阶段 | 说明 |
|---|---|---|---|---|
| M01 | 仓库打开模块 | ✅ | 第1阶段 | 打开/关闭/最近仓库 |
| M02 | 仓库初始化模块 | ✅ | 第1阶段 | `git init` |
| M03 | 状态读取模块 | ✅ | 第1阶段 | `git status` 完整读取 |
| M04 | 暂存区模块 | ✅ | 第1阶段 | Stage/Unstage/Discard |
| M05 | 提交模块 | ✅ | 第1阶段 | 撰写消息并提交 |
| M06 | Diff 查看模块 | ✅ | 第1阶段 | 文本文件 unified diff |
| M07 | 分支管理模块 | ✅ | 第1阶段 | 列表/切换/新建/删除 |
| M08 | 提交历史模块 | ✅ | 第1阶段 | 列表 + 分页 + 详情 |
| M09 | 后台刷新模块 | ✅ | 第1阶段 | 文件系统监听 + 事件推送 |
| M10 | 设置模块 | ✅ | 第1阶段 | 用户名/邮箱/主题 |
| M11 | 错误处理模块 | ✅ | 第1阶段 | 全局 Toast + 错误边界 |
| M12 | 搜索/过滤模块 | ❌ | 第2阶段 | commit 搜索、文件过滤 |
| M13 | 提交图谱模块 | ❌ | 第2阶段 | 分支拓扑 DAG 渲染 |
| M14 | 并排 Diff 模块 | ❌ | 第2阶段 | Side-by-side diff |
| M15 | 语法高亮模块 | ❌ | 第2阶段 | Diff 区语法高亮 |
| M16 | Stash 模块 | ❌ | 第2阶段 | Stash/Pop/Drop |
| M17 | Tag 管理模块 | ❌ | 第2阶段 | 标签列表/创建/删除 |
| M18 | GitHub OAuth 模块 | ❌ | 第2阶段 | Device Flow 登录 |
| M19 | Remote 操作模块 | ❌ | 第2阶段 | Push/Pull/Fetch |
| M20 | PR 管理模块 | ❌ | 第3阶段 | PR 列表/创建/审查 |
| M21 | Merge/Rebase 模块 | ❌ | 第3阶段 | 合并/变基操作 |
| M22 | 冲突解决模块 | ❌ | 第3阶段 | 三路合并视图 |
| M23 | Blame 模块 | ❌ | 第3阶段 | 行注释追踪 |
| M24 | 插件系统 | ❌ | 第3阶段 | 可扩展插件接口 |

---

## 2. MVP 模块依赖关系

```
M01 仓库打开
  └─► M03 状态读取
        ├─► M04 暂存区
        │     └─► M05 提交
        ├─► M06 Diff 查看
        └─► M09 后台刷新

M07 分支管理（依赖 M01）
M08 提交历史（依赖 M01）
  └─► M06 Diff 查看（复用）

M10 设置（独立模块）
M11 错误处理（横切关注点，所有模块使用）
```

---

## 3. 各模块详细设计

### M01 — 仓库打开模块

**Rust 端**：
- `open_repository(path)` → 验证路径是 git 仓库 → 创建 `GitService` 实例 → 注册到 `AppState`
- `close_repository(path)` → 从 `AppState` 移除 → 停止文件监听
- `get_recent_repos()` → 从持久化配置读取最近列表
- `init_repository(path)` → `git2::Repository::init()`

**前端**：
- 欢迎页：最近仓库列表（卡片样式）+ 打开按钮 + 新建按钮
- 文件选择对话框（Tauri `dialog.open()`）
- 打开成功后跳转到主工作区页面

**状态**：
```typescript
// repoStore.ts
currentRepoPath: string | null
recentRepos: RecentRepo[]
```

---

### M03 — 状态读取模块

**Rust 端**：
- `get_status(repo_path)` → `git2::Repository::statuses()` → 构建 `StatusSummary`
- 性能考量：`StatusOptions` 配置 `include_untracked(true)` + `recurse_untracked_dirs(true)`（必要时关闭递归优化性能）

**前端**：
- `useGitStatus(repoPath)` hook：TanStack Query，staleTime 3s
- 状态变化通过 `repo:changed` 事件触发重新获取
- 文件列表按"已暂存"/"未暂存"/"未跟踪"三组展示

---

### M04 — 暂存区模块

**Rust 端**：
- `stage_files(repo_path, paths[])` → `repo.index().add_path()`（对已修改/新文件）
- `unstage_files(repo_path, paths[])` → `repo.reset_default(HEAD, paths[])` 
- `discard_changes(repo_path, paths[])` → `repo.checkout_head()` with path spec（危险操作，需确认）
- `stage_all(repo_path)` → 批量 stage 所有未暂存文件

**前端**：
- `StagingPanel`：两列（已暂存/未暂存），每行文件 + 状态徽章 + 操作按钮
- 快捷操作：全部暂存 / 全部取消暂存
- 点击文件 → 更新选中状态 → 触发 diff 查看（通过 Zustand）
- `useStaging` hook：封装 mutation，成功后自动失效状态缓存

---

### M05 — 提交模块

**Rust 端**：
- `commit_changes(repo_path, message)` → 读取 git config 获取 author → `repo.commit()`
- 校验：消息不为空、有已暂存文件
- 返回新的 `CommitSummary`（供前端展示）

**前端**：
- `CommitMessageEditor`：多行文本输入，subject（第一行）+ body（剩余）
- 字符计数（subject 建议 < 72 字符）
- 快捷键：`Ctrl+Enter` 提交
- 提交后：清空输入框、刷新状态、刷新历史

---

### M06 — Diff 查看模块

**Rust 端**：
- `get_diff(repo_path, DiffParams)` → 根据类型选择不同 git2 diff API：
  - 工作区 vs 索引（未暂存 diff）：`diff_index_to_workdir()`
  - 索引 vs HEAD（已暂存 diff）：`diff_head_to_index()`
  - commit diff：`diff_tree_to_tree()`
- 构建 `FileDiff`（hunks + lines）
- 大文件截断：diff > 500KB 或 > 5000 行时截断

**前端**：
- `DiffViewer` 组件（见 G 章节详细设计）
- 右侧面板根据选中文件类型（暂存/未暂存/历史）切换 diff 来源
- 二进制文件显示占位符（文件大小、类型）

---

### M07 — 分支管理模块

**Rust 端**：
- `get_branches(repo_path)` → 列举本地 + 远程分支
- `checkout_branch(repo_path, name)` → `git2::Repository::set_head()` + `checkout_head()`
  - 工作区有修改时：返回 `CHECKOUT_CONFLICT` 错误（不强制切换）
- `create_branch(repo_path, name, from?)` → `repo.branch()`
- `delete_branch(repo_path, name)` → 检查是否已合并 → `Branch::delete()`

**前端**：
- 侧边栏分支列表：当前分支高亮 + 切换按钮
- 分支右键菜单：新建/删除/重命名
- 切换分支时，工作区有修改则弹出确认对话框

---

### M08 — 提交历史模块

**Rust 端**：
- `get_commit_log(repo_path, CommitLogParams{ page, limit, branch? })` → `repo.revwalk()` 分页
- `get_commit_detail(repo_path, sha)` → 获取完整 commit 信息 + changed_files

**前端**：
- 虚拟列表渲染（`@tanstack/react-virtual`）
- 每行：短 SHA（7位）+ commit 消息 + 作者 + 相对时间
- 分支标签（refs decorations）用色块显示
- 无限滚动加载（TanStack Query `useInfiniteQuery`）
- 点击 commit → 右侧显示 `CommitDetail`

---

### M09 — 后台刷新模块

**Rust 端**：
- 使用 `notify` crate 监听 `.git` 目录变化
- debounce 500ms（避免大量文件操作时洪水）
- 变化时 emit `repo:changed` 事件

**前端**：
- `useRepoEvents` hook：监听 `repo:changed`，失效相关 Query 缓存
- 不做轮询，完全事件驱动

---

### M10 — 设置模块

**配置项（MVP）**：
- Git 用户名（user.name）
- Git 邮箱（user.email）
- 主题（暗色/亮色）
- 字体大小（12/13/14px）

**存储**：
- Git 配置读写通过 `git config --global`（通过 git2 `Config` API）
- 应用设置存储在 `~/.config/vibegit/settings.json`

---

## 4. 横切关注点（所有模块共用）

### M11 — 错误处理模块

```typescript
// 统一错误处理流程
invoke('some_command')
  → services 层捕获错误 → parseApiError()
  → hooks 层 onError 回调 → useToastStore.addToast()
  → Toaster 组件显示 Toast
```

### 后台刷新（横切）

所有数据变更操作（stage、commit、checkout）完成后：
```typescript
queryClient.invalidateQueries({ queryKey: ['repo', repoPath] })
// 触发所有依赖该仓库数据的 Query 重新获取
```

---

## 5. 模块开发优先级排序

```
Phase 1 — 骨架（第0阶段）：
  M01 仓库打开 → M11 错误处理 → 基础 UI 布局

Phase 2 — 核心工作流（第1阶段前期）：
  M03 状态读取 → M06 Diff 查看 → M04 暂存区 → M05 提交

Phase 3 — 补充功能（第1阶段后期）：
  M07 分支管理 → M08 提交历史 → M09 后台刷新 → M10 设置

Phase 4 — 增强（第2阶段）：
  M12 搜索 → M13 提交图谱 → M15 语法高亮 → M18 GitHub OAuth → M19 远程操作
```
