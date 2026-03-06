# N. Codex 多 Agent 并行开发指南

> **本章回答三个问题**：  
> 1. 如何用分支方式让多个 Codex Agent 并行开发、最后合并？  
> 2. 这个项目最适合拆成几个 Agent？  
> 3. 每个 Agent 的指令怎么写？

---

## 1. 核心原则：先串行建好地基，再并行盖房子

多 Agent 并行开发的关键矛盾是**依赖关系**。如果 Agent A 还没写完 `domain/models.rs`，Agent B 就无法正确引用类型，最终合并时会冲突。

**正确姿势**：

```
Phase 0（单 Agent 串行）
   ↓ 合并后，才允许多 Agent 并行
Phase 1.1（两个 Agent 并行）
   ↓ 合并后，才允许下一批并行
Phase 1.2（两个 Agent 并行）
   ↓ 合并后
Phase 2（两个 Agent 并行）
```

每一层合并完成后，才开启下一层的并行。**不要在还没合并的分支上再开新分支**（菱形合并极易产生大面积冲突）。

---

## 2. 哪些文件会冲突——冲突高危区

在安排并行 Agent 时，必须让每个 Agent 操作**互不重叠**的文件集合：

| 高危文件（极易冲突） | 解决策略 |
|---|---|
| `src-tauri/src/lib.rs` | 由 Phase 0 Agent 写完注册框架，后续 Agent 只追加注册项到指定区域 |
| `src-tauri/src/domain/models.rs` | Phase 0 Agent 负责全量定义，后续 Agent 只读不写 |
| `src/types/bindings.ts` | 自动生成文件，每次改 Rust 类型后重新生成，不手写 |
| `src/App.tsx` | 路由注册文件，每个 Agent 各自写 page 组件，最后合并时手动加路由 |
| `package.json` / `Cargo.toml` | 各 Agent 不增加新依赖，依赖需求提前在本文档 Section 8 中列全 |

**低冲突区（可放心并行）**：
- `src/components/git/` 中的各独立组件文件
- `src/hooks/` 中的各独立 hook 文件
- `src-tauri/src/commands/` 中的各独立 command 文件
- `src-tauri/src/services/git_service.rs` 的各独立方法（Agent 之间按方法分段，不重叠）

---

## 3. 分支命名规范

```
主干：    main
准备层：  setup/phase-0           ← Phase 0 完成后合并到 main
并行层一：feat/rust-core           ← 从 main(phase-0 合并后) 切出
          feat/frontend-core       ← 从 main(phase-0 合并后) 切出
并行层二：feat/branch-history      ← 从 main(层一合并后) 切出
          feat/watcher-settings    ← 从 main(层一合并后) 切出
并行层三：feat/github-integration  ← 从 main(层二合并后) 切出
          feat/commit-graph        ← 从 main(层二合并后) 切出
```

**创建分支前，先确认基准 commit**：

```bash
# 查看当前 main 的 HEAD
git log main --oneline -3

# 从正确的 main 点切出新分支
git checkout main
git pull origin main
git checkout -b feat/rust-core
```

---

## 4. Agent 数量与分配——推荐方案（6 个 Agent）

下表是针对 VibeGit 项目的最优 Agent 分配：

| Agent 编号 | 分支名 | 负责模块 | 依赖前置 | 可并行伙伴 |
|---|---|---|---|---|
| **Agent 0** | `setup/phase-0` | 工程初始化、类型定义、目录结构、CSS 变量 | 无 | 无（串行） |
| **Agent A** | `feat/rust-core` | Rust GitService 所有方法、domain 模型、全部 commands | Agent 0 完成 | Agent B |
| **Agent B** | `feat/frontend-core` | 前端所有组件、hooks、store、services（使用 mock 数据） | Agent 0 完成 | Agent A |
| **Agent C** | `feat/branch-history` | 分支管理模块（M07）+ 提交历史模块（M08） | A + B 合并 | Agent D |
| **Agent D** | `feat/watcher-settings` | 后台刷新模块（M09）+ 设置页面（M10）+ Toast 系统（M11） | A + B 合并 | Agent C |
| **Agent E** | `feat/github-integration` | GitHub OAuth（M18）+ Remote 操作 Push/Pull/Fetch（M19） | C + D 合并 | Agent F |
| **Agent F** | `feat/commit-graph` | 提交图谱 DAG（M13）+ Diff 语法高亮（M15）+ 并排 Diff（M14） | C + D 合并 | Agent E |

> **精简版**（资源有限时只用 4 个）：合并 Agent C+D 为一个，合并 Agent E+F 为一个。

---

## 5. 合并流程与 PR 策略

### 5.1 合并顺序

```
Agent 0 PR → 合并到 main
    ↓
Agent A PR + Agent B PR → 分别审查 → 先合 A，再合 B（B 合并前需 rebase onto A）
    ↓
Agent C PR + Agent D PR → 分别审查 → 先合 C，再合 D
    ↓
Agent E PR + Agent F PR → 分别审查 → 任意顺序
```

### 5.2 同批 Agent 合并时的 rebase 步骤

假设 Agent A 的 PR 先被合并，Agent B 需要在合并前做：

```bash
# 在 Agent B 的分支上
git fetch origin
git rebase origin/main          # 将 B 的提交接到最新 main 上
# 解决冲突（主要是 App.tsx 的路由和 Cargo.toml 的依赖）
git push --force-with-lease origin feat/frontend-core
```

### 5.3 合并检查清单

合并前，由 PR 审查者确认：

- [ ] `cargo build` 通过（无编译错误）
- [ ] `pnpm build` 通过（无 TypeScript 错误）
- [ ] `bindings.ts` 与 `domain/models.rs` 同步（重新运行 `cargo tauri dev` 验证）
- [ ] 无新增 `todo!()` 残留（Phase 0 的占位可以，后续阶段不允许）
- [ ] 没有新增全局状态污染（Zustand store 没有未清理的副作用）

---

## 6. 每个 Agent 的详细指令

以下是给每个 Agent 写的"任务单"，可直接粘贴给 Codex：

---

### Agent 0 — 工程初始化

**分支**：`setup/phase-0`  
**基于**：`main`（空仓库）

**任务**：

```
你是一个 Tauri 2 + Rust + React 工程师，负责搭建 VibeGit 项目骨架。

请严格按照以下顺序执行，每步完成后验证，再进行下一步：

1. 使用 `pnpm create tauri-app` 创建工程，模板选 react-ts，包管理器选 pnpm。

2. 安装前端依赖（详见 docs/architecture/01-tech-selection.md Section 3）：
   - tailwindcss + postcss + autoprefixer
   - shadcn/ui（init，选择 Dark 主题，CSS variables）
   - zustand + @tanstack/react-query + motion + lucide-react
   - clsx + tailwind-merge
   - @tauri-apps/plugin-dialog + @tauri-apps/plugin-fs

3. 在 `src-tauri/Cargo.toml` 添加 Rust 依赖（详见 docs/architecture/01-tech-selection.md Section 4）：
   git2、serde、serde_json、tokio、thiserror、anyhow、specta、tauri-specta、
   tracing、tracing-subscriber、notify、keyring、reqwest、uuid、tauri-plugin-dialog、
   tauri-plugin-fs、tauri-plugin-shell

4. 完整定义 `src-tauri/src/domain/models.rs`（详见 docs/architecture/05-domain-models.md）。
   必须包含所有字段，不能省略。添加 `#[derive(Debug, Serialize, Deserialize, Clone, specta::Type)]`。

5. 完整定义 `src-tauri/src/domain/error.rs`（详见 docs/architecture/05-domain-models.md 错误类型部分）。

6. 配置 specta 类型导出（`lib.rs`），确保运行 `cargo tauri dev` 后 `src/types/bindings.ts` 自动生成。

7. 按照 docs/architecture/04-directory-structure.md 创建所有目录和空文件（`// TODO` 占位）。

8. 在 `src/styles/globals.css` 建立 CSS 变量系统（详见 docs/architecture/07-frontend-architecture.md Section 8）。

9. 搭建 AppLayout、TitleBar、欢迎页骨架（仅布局和占位符，不实现逻辑）。

10. 配置 ESLint + Prettier + cargo fmt。

验证：
- `cargo build` 无报错
- `pnpm build` 无报错
- `bindings.ts` 文件已生成且内容非空
- 截图窗口确认暗色主题渲染正常

不要实现任何 Git 功能，只做骨架。
```

---

### Agent A — Rust 核心后端

**分支**：`feat/rust-core`  
**基于**：`main`（Phase 0 合并后）

**任务**：

```
你是一个 Rust + git2 工程师，负责实现 VibeGit 的 Rust 后端所有 Git 功能。

前提：工程骨架已由 Agent 0 搭建完成，domain/models.rs 和 domain/error.rs 已完整定义。
你不需要修改这两个文件（只读）。

请按以下顺序实现（每步实现完后运行 cargo build 验证）：

1. 实现 `services/git_service.rs` 的 GitService 结构体（详见 docs/architecture/06-rust-backend.md Section 3）：
   按照以下方法顺序实现，每个方法完整实现（不用 todo!()）：
   - open() ← 第一个，最重要
   - get_summary()
   - get_status()
   - get_diff()
   - stage_files() ← 注意 index.write() 必须调用，注意已删除文件用 remove_path
   - unstage_files()
   - discard_changes()
   - commit()

2. 实现 `services/repo_manager.rs`（最近仓库列表，持久化到 config.json）。

3. 实现 `state.rs`（AppState，参见 docs/architecture/06-rust-backend.md Section 2）。

4. 实现 `commands/repo.rs`：open_repository、close_repository、get_recent_repos、init_repository。

5. 实现 `commands/git.rs`：get_repo_summary、get_status、stage_files、unstage_files、
   discard_changes、commit_changes。

6. 实现 `commands/diff.rs`：get_diff、get_file_diff。

7. 在 `lib.rs` 中注册上述所有 commands（使用占位注释标记注册区域，便于后续 Agent 追加）：
   // === COMMANDS START ===
   // === COMMANDS END ===

8. 实现 `utils/path.rs`（路径规范化）和 `utils/encoding.rs`（文件编码检测）。

注意事项（来自 docs/architecture/13-codex-instructions.md Section 9）：
- git2::Repository 不是 Send，必须用 Mutex<Repository> 包装
- 每个 async 方法中锁的持有时间要尽量短，不要跨 await 持有
- 耗时操作用 tokio::task::spawn_blocking

验证：
- cargo build 无报错
- 在真实 git 仓库上运行 open_repository，控制台打印 RepositorySummary
- 完成一遍 stage → commit 流程

不要修改任何前端文件。不要实现 branch 相关命令（留给 Agent C）。
不要实现 GitHub/OAuth 相关（留给 Agent E）。
```

---

### Agent B — 前端核心组件

**分支**：`feat/frontend-core`  
**基于**：`main`（Phase 0 合并后）

**任务**：

```
你是一个 React + TypeScript + Tailwind CSS 前端工程师，负责实现 VibeGit 的前端所有核心 UI 和逻辑。

前提：工程骨架已由 Agent 0 搭建，bindings.ts 已生成。
重要：Rust 后端尚未实现，你必须使用 Mock 数据开发所有前端功能。

Mock 策略：在 `src/services/` 的每个 API 文件顶部添加 `USE_MOCK` 常量，
当为 true 时返回 mock 数据，当为 false 时调用真实 invoke()。

请按顺序实现：

1. 实现 `src/store/`（repoStore、uiStore、toastStore）（详见 docs/architecture/07-frontend-architecture.md）。

2. 实现 `src/lib/`（cn.ts、format.ts、diff.ts、constants.ts）。

3. 实现 `src/services/`（repoApi.ts、gitApi.ts、diffApi.ts），包含 mock 数据。
   Mock 数据要真实可用（真实的文件路径、diff 内容、commit SHA 等）。

4. 实现 `src/hooks/`（useRepo、useGitStatus、useDiff、useStaging、useCommit、useToast、useRepoEvents、useKeyboard）。

5. 实现 `src/components/shared/`（EmptyState、ErrorBoundary、LoadingSpinner、ResizablePanel、VirtualList）。

6. 实现 `src/components/git/` 的所有组件：
   - FileStatusBadge（文件状态徽章：M/A/D/R，对应颜色）
   - FileChangeList（文件变更列表，支持点击选中）
   - StagingPanel（两列：已暂存/未暂存 + 全部暂存/取消按钮）
   - CommitMessageEditor（主题行 + 正文，Ctrl+Enter 提交，字符计数）
   - DiffHunk（单个 hunk 渲染，绿色/红色行）
   - DiffViewer（接受 FileDiff prop，不知道数据来源）
   - StatusBar（底部状态栏，显示分支名、文件统计）

7. 实现 `src/pages/Welcome.tsx`（最近仓库卡片列表 + 打开按钮）。

8. 实现 `src/pages/Repository.tsx`（三栏布局：StagingPanel + DiffViewer + 右侧操作）。

9. 实现全局 Toast 系统（Toaster 组件 + useToast hook）。

10. 在 `src/App.tsx` 设置路由（Welcome ↔ Repository）。

样式规范（详见 docs/architecture/07-frontend-architecture.md 和 08-hidpi-ui.md）：
- 全程使用 CSS 变量（var(--background) 等），不硬编码颜色
- 字体：Inter Variable（正文）、JetBrains Mono（代码/diff）
- 使用 clsx + tailwind-merge（cn() 工具）组合 class
- 组件不超过 150 行，超过则拆子组件

验证：
- pnpm build 无报错
- 截图欢迎页和主工作区页面，确认布局和暗色主题正常
- Mock 模式下能完整点击 stage → commit 流程（数据变化正常）

不要修改任何 Rust 文件。不要实现分支列表 UI（留给 Agent C）。
不要实现 GitHub 相关 UI（留给 Agent E）。
```

---

### Agent C — 分支管理 + 提交历史

**分支**：`feat/branch-history`  
**基于**：`main`（Agent A + Agent B 合并后）

**任务**：

```
你是 VibeGit 的功能开发工程师，负责实现分支管理模块（M07）和提交历史模块（M08）。

前提：Rust 核心后端和前端骨架均已完成并合并。

Rust 端（在已有的 GitService 上追加方法）：

1. 在 `services/git_service.rs` 中追加：
   - get_branches() → Vec<BranchInfo>（本地+远程）
   - checkout_branch(name) → 注意工作区有修改时返回 CHECKOUT_CONFLICT 错误
   - create_branch(name, from?) → BranchInfo
   - delete_branch(name) → 检查已合并才允许删除
   - get_commit_log(params: CommitLogParams) → CommitLogResult（分页，见 13-codex-instructions.md Section 9）
   - get_commit_detail(sha) → CommitDetail

2. 在 `commands/branch.rs` 中实现：
   get_branches、checkout_branch、create_branch、delete_branch

3. 在 `commands/history.rs` 中实现：
   get_commit_log、get_commit_detail

4. 将新 commands 追加注册到 lib.rs 的 // === COMMANDS START/END === 区域内。

前端端：

5. 实现 `src/services/branchApi.ts` 和 `src/services/historyApi.ts`（含 mock 数据）。

6. 实现 `src/hooks/useBranches.ts`、`useCommitLog.ts`、`useCommitDetail.ts`。

7. 实现 `src/components/git/BranchSelector.tsx`（当前分支 + 下拉切换 + 创建/删除分支对话框）。
   切换分支时，如果工作区有修改，弹出确认对话框（使用 shadcn/ui Dialog）。

8. 实现 `src/components/git/CommitList.tsx`（虚拟列表，@tanstack/react-virtual）。
   每行：7位短 SHA + commit 消息 + 作者头像 + 相对时间 + refs 标签色块。
   无限滚动（TanStack Query useInfiniteQuery）。

9. 实现 `src/components/git/CommitDetail.tsx`（commit 元数据 + 变更文件列表 + 复用 DiffViewer）。

10. 将分支列表和提交历史接入 Repository 页面左侧边栏。

验证：
- cargo build 无报错，pnpm build 无报错
- 能切换分支，切换后状态面板内容更新
- 提交历史列表可滚动加载，点击 commit 能看到 diff
- 在工作区有修改时切换分支，确认对话框弹出
```

---

### Agent D — 后台刷新 + 设置 + 错误处理完善

**分支**：`feat/watcher-settings`  
**基于**：`main`（Agent A + Agent B 合并后）

**任务**：

```
你是 VibeGit 的功能开发工程师，负责实现后台刷新模块（M09）、设置页面（M10）和完善全局错误处理（M11）。

Rust 端：

1. 实现 `services/watcher.rs`（详见 docs/architecture/09-module-breakdown.md Section M09）：
   - 使用 notify crate 监听 .git 目录
   - debounce 500ms
   - 变化时 emit repo:changed 事件（携带仓库路径）
   - 在 open_repository 时启动监听，close_repository 时停止

2. 实现 git config 读写（user.name、user.email）并添加到 GitService：
   get_git_config(key) → String
   set_git_config(key, value) → ()

3. 实现应用设置持久化（~/.config/vibegit/settings.json）：
   存储：主题偏好、字体大小

4. 在 commands/git.rs 中添加 get_git_config、set_git_config 命令并注册。

前端端：

5. 实现 `src/hooks/useRepoEvents.ts`：
   监听 repo:changed 事件，失效对应仓库的所有 Query 缓存（见 13-codex-instructions.md Section 9）。

6. 完善全局 Toast 系统（所有错误统一通过 parseApiError + useToast 显示）：
   - 解析 AppError 的 code 字段，映射为用户可读的中文提示
   - 区分 warning（黄色）和 error（红色）
   - 自动消失（3秒），支持手动关闭

7. 实现 `src/pages/Settings.tsx`：
   - Git 用户名/邮箱（读取 git config，可编辑保存）
   - 主题切换（暗色/亮色）
   - 字体大小选择（12/13/14px）
   - 设置保存后即时生效

8. 确保所有 Mutation 操作（stage、commit、checkout）的错误都通过 Toast 显示。

验证：
- 在另一个终端执行 git commit 后，VibeGit UI 在 1 秒内自动刷新
- 打开设置页，修改用户名后执行 commit，提交作者信息正确
- 触发一个错误（如 stage 不存在的文件），Toast 提示友好可读
```

---

### Agent E — GitHub 集成

**分支**：`feat/github-integration`  
**基于**：`main`（Agent C + Agent D 合并后）

**任务**：

```
你是 VibeGit 的 GitHub 集成工程师，负责实现 GitHub OAuth（M18）和 Remote 操作（M19）。

Rust 端：

1. 实现 `services/auth_service.rs`（GitHub Device Flow OAuth）：
   - start_device_auth() → 返回 user_code 和 verification_uri（前端展示给用户）
   - poll_device_auth() → 轮询 GitHub API 获取 access_token
   - 使用 keyring 持久化 token（Windows Credential Manager / macOS Keychain）
   - get_auth_state() → Option<GitHubUser>
   - logout() → 清除 keyring 中的 token

2. 实现 `services/github_api.rs`：
   - get_user() → GitHubUser（验证 token 有效性）
   - list_user_repos() → Vec<RepoSummary>

3. 实现 fetch / push / pull 操作（在 GitService 中）：
   - 使用 tokio::task::spawn_blocking 包装（网络操作）
   - 通过 tauri window.emit() 发送进度事件（task:progress）
   - HTTPS 凭证通过 auth_service 的 token 注入 git2 RemoteCallbacks

4. 实现相关 commands：auth::github_start_auth、github_poll_auth、github_logout、
   github_get_user；git::fetch_remote、push_branch、pull_branch

前端端：

5. 实现 GitHub OAuth 登录界面（弹窗：展示 user_code + 打开浏览器按钮 + 轮询等待）。

6. 实现 Push/Pull/Fetch 按钮（含进度条，监听 task:progress 事件）。

7. 在 StatusBar 中显示 GitHub 登录状态（头像 + 用户名 / 未登录）。

安全注意事项：
- token 只存 keyring，不存 JSON 文件，不写日志
- push 前检查是否有 upstream 分支，无则提示设置

验证：
- 完成 Device Flow 登录，控制台打印 GitHub 用户名
- 对一个 GitHub 仓库执行 push，进度条正常显示，远端能看到新 commit
```

---

### Agent F — 提交图谱 + 高级 Diff

**分支**：`feat/commit-graph`  
**基于**：`main`（Agent C + Agent D 合并后）

**任务**：

```
你是 VibeGit 的可视化工程师，负责实现提交图谱（M13）、Diff 语法高亮（M15）和并排 Diff（M14）。

提交图谱（M13）：

1. Rust 端：在 get_commit_log 的返回数据中追加 graph_node 字段：
   每个 commit 包含其在图谱中的列位置（column）和连线信息（parents_columns）。
   使用拓扑排序 + 列分配算法（贪心：尽量复用空闲列）。

2. 前端端：在 CommitList 的每行左侧，用 SVG 渲染分支连线：
   - 垂直线：当前列的连续提交
   - 分叉线：分支创建点
   - 合并线：merge commit
   - 每个分支用不同颜色（从 CSS 变量调色板中取）
   - 大仓库（> 1000 提交）使用虚拟列表裁剪，图谱线只渲染可见区域

语法高亮（M15）：

3. 集成 shiki（WebAssembly 版本，离线可用）：
   - 支持的语言：ts/js/tsx/jsx/rs/py/go/java/c/cpp/json/yaml/toml/md/css/html/sh
   - 懒加载：只在 DiffViewer 首次显示时初始化 shiki
   - 为 DiffHunk 中的每行代码添加语法高亮

并排 Diff（M14）：

4. 实现 SideBySideDiff 组件（DiffViewer 的替代模式）：
   - 左：删除行（红色背景）；右：新增行（绿色背景）
   - 上下同步滚动
   - 在 DiffViewer 右上角添加 unified/side-by-side 切换按钮

验证：
- 打开一个有多条分支的仓库，提交图谱连线正确显示
- Diff 中的 TypeScript 代码有语法高亮（关键字蓝色/字符串绿色等）
- 并排 Diff 模式下左右滚动同步
```

---

## 7. 合并时的冲突解决手册

### 7.1 `src/App.tsx` 路由冲突

两个 Agent 都可能追加路由。解决方式：手动合并，保留两份路由项：

```tsx
// 合并后的 App.tsx
<Routes>
  <Route path="/" element={<Welcome />} />
  <Route path="/repo/:repoPath" element={<Repository />} />
  <Route path="/settings" element={<Settings />} />  {/* Agent D 添加 */}
</Routes>
```

### 7.2 `lib.rs` 命令注册冲突

利用注释标记区域，避免冲突：

```rust
// === COMMANDS START ===
// repo commands (Agent 0)
repo::open_repository,
repo::close_repository,
// git commands (Agent A)
git::get_status,
git::stage_files,
// branch commands (Agent C) — 追加在此
branch::get_branches,
// history commands (Agent C) — 追加在此
history::get_commit_log,
// === COMMANDS END ===
```

### 7.3 `Cargo.toml` 依赖冲突

各 Agent 不应自行添加依赖。如确实需要，在 PR 描述中注明新增依赖，由审查者手动合并。

### 7.4 `bindings.ts` 冲突

该文件由 specta 自动生成，**不要手动编辑**。合并后重新运行一次 `cargo tauri dev` 即可重新生成正确的版本。

---

## 8. 各阶段预装依赖清单（Phase 0 一次性装完）

Agent 0 在初始化时统一安装以下所有依赖，后续 Agent 不再新增：

**前端（pnpm）**：
```
tailwindcss postcss autoprefixer
@shadcn/ui（via dlx init）
zustand
@tanstack/react-query
@tanstack/react-virtual
motion
lucide-react
clsx
tailwind-merge
@tauri-apps/plugin-dialog
@tauri-apps/plugin-fs
@tauri-apps/plugin-shell
shiki                            ← Agent F 会用
```

**Rust（Cargo.toml）**：
```toml
[dependencies]
git2 = { version = "0.19", features = ["vendored-libgit2"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
thiserror = "1"
anyhow = "1"
specta = { version = "2", features = ["derive"] }
tauri-specta = { version = "2", features = ["derive", "typescript"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
notify = "6"
notify-debouncer-mini = "0.4"
keyring = "2"
reqwest = { version = "0.12", features = ["json", "rustls-tls"], default-features = false }
uuid = { version = "1", features = ["v4"] }
tauri-plugin-dialog = "2"
tauri-plugin-fs = "2"
tauri-plugin-shell = "2"
```

---

## 9. 快速参考：给 Codex 的通用开场提示

在每次给 Codex Agent 发任务前，加上这段前置说明：

```
你正在开发 VibeGit 项目（一个 Tauri 2 + Rust + React 的可视化 Git 桌面工具）。
完整架构文档在 docs/architecture/ 目录下（00-overview.md 到 14-parallel-agents.md）。
开始前，请先阅读 00-overview.md 了解项目整体，再阅读与你任务相关的章节。

你的任务范围在 [分支名] 分支上。请严格限制在任务范围内，不要修改其他模块的代码。
完成后运行 cargo build 和 pnpm build 确认无报错，再提交 PR。
```
