# M. Codex 可执行输出 — 给 AI 的工程实施指令

> **本章是给 Codex/AI 开发助手的直接工程实施指令。**  
> 按照本章的指导顺序执行，可以避免过度耦合、避免推倒重来、确保可持续迭代。

---

## 1. 总体原则

1. **按阶段执行**，不要一次性写完所有代码
2. **先定义类型，再写实现**：所有数据结构先定义好，再写逻辑
3. **先打通主干，再完善细节**：先让 invoke 调用链路跑通，再做 UI 美化
4. **Mock 优先**：前端开发时先 mock Rust 端数据，不要等 Rust 实现完再开始
5. **每个模块可独立运行**：组件、hooks、services 各自可独立测试
6. **不要提前优化**：虚拟列表、语法高亮等优化放在功能完成后再加

---

## 2. Phase 0：初始化工程（第一步执行）

### Step 0-1：创建 Tauri + React 工程

```bash
# 使用官方脚手架
pnpm create tauri-app vibegit \
  --template react-ts \
  --manager pnpm

cd vibegit
pnpm install
```

### Step 0-2：安装前端依赖

```bash
pnpm add tailwindcss postcss autoprefixer
pnpm dlx tailwindcss init -p

# shadcn/ui 初始化
pnpm dlx shadcn@latest init
# 选择：Dark 主题，CSS variables，src/components/ui 目录

# 其他依赖
pnpm add zustand @tanstack/react-query motion lucide-react clsx tailwind-merge
pnpm add @tauri-apps/plugin-dialog @tauri-apps/plugin-fs
```

### Step 0-3：安装 Rust 依赖

在 `src-tauri/Cargo.toml` 中添加（见 01-tech-selection.md 第 4 节）。

### Step 0-4：先定义这些 Rust 类型（`domain/models.rs`）

按照 05-domain-models.md 完整定义所有模型，**不要跳过任何字段**。这是后续所有工作的基础。

```
优先级顺序：
1. AppError（所有类型先有错误类型）
2. RepositorySummary + HeadState
3. StatusSummary + WorkingDirEntry + ChangeStatus
4. BranchInfo
5. CommitSummary + CommitDetail
6. FileDiff + DiffHunk + DiffLine
```

### Step 0-5：配置 specta 类型生成

在 `lib.rs` 中配置好类型导出，确保 `src/types/bindings.ts` 可以自动生成。

**验证**：运行 `cargo tauri dev`，确认 `bindings.ts` 正确生成。

### Step 0-6：搭建前端目录结构

按照 04-directory-structure.md 第 1 节创建所有目录和空文件（`// TODO` 占位即可）。

### Step 0-7：建立 CSS 变量系统

在 `src/styles/globals.css` 中定义暗色主题 CSS 变量（见 07-frontend-architecture.md 第 8 节）。

**验证**：截图确认暗色主题渲染正常，字体清晰。

---

## 3. Phase 1.1：核心工作流实施

### 执行顺序

#### Step 1-1：实现 GitService 骨架

```rust
// 先写空实现，只返回 todo!() 或 mock 数据
impl GitService {
    pub fn open(path: &str) -> Result<Self, AppError> { todo!() }
    pub async fn get_summary(&self) -> Result<RepositorySummary, AppError> { todo!() }
    pub async fn get_status(&self) -> Result<StatusSummary, AppError> { todo!() }
    // ... 其他方法全部 todo!()
}
```

**目的**：让 Rust 代码可以编译，占位接口确定。

#### Step 1-2：实现 open_repository 命令（端到端打通）

这是**第一个要完全实现**的 command。打通整个 IPC 链路：
```
前端 invoke → Rust command → GitService::open → 返回 RepositorySummary → 前端显示
```

实现步骤：
1. `GitService::open()` 实际实现（git2 打开仓库）
2. `GitService::get_summary()` 实际实现
3. `commands/repo.rs` `open_repository` command
4. `src/services/repoApi.ts` 封装 invoke
5. `src/hooks/useRepo.ts` 封装 hook
6. 简单欢迎页面，能调用打开仓库

**验证**：能打开一个真实 git 仓库，控制台看到 RepositorySummary。

#### Step 1-3：实现 get_status

1. `GitService::get_status()` 实现（git2 statuses）
2. `commands/git.rs` `get_status` command
3. `src/services/gitApi.ts`
4. `src/hooks/useGitStatus.ts`
5. `StagingPanel` 组件（先只展示文件列表，不做操作）

**验证**：打开仓库后能看到修改的文件列表。

#### Step 1-4：实现 get_diff

1. `GitService::get_diff()` 实现（git2 diff API）
2. `commands/diff.rs` `get_diff` command
3. `src/services/diffApi.ts`
4. `src/hooks/useDiff.ts`
5. `DiffViewer` + `DiffHunk` 组件（先不加语法高亮）

**验证**：点击文件能看到 diff 内容。

#### Step 1-5：实现 stage / unstage / commit

按照以下顺序，每个都是"实现 service → 实现 command → 实现 hook → 接入 UI"：
1. `stage_files`
2. `unstage_files`  
3. `discard_changes`（加确认对话框）
4. `commit_changes`（加 CommitMessageEditor 组件）

**验证**：完整走一遍 stage → commit 流程，git log 能看到新提交。

---

## 4. Phase 1.2：补全 MVP

### Step 2-1：分支管理

实现顺序：`get_branches` → `checkout_branch` → `create_branch` → `delete_branch`

**注意**：checkout 时如果工作区有修改，必须先提示用户，不要静默失败。

### Step 2-2：提交历史

1. 实现 `get_commit_log`（分页，每次 50 条）
2. 实现 `get_commit_detail`
3. `CommitList` 组件（先不用虚拟列表，等有性能问题再加）
4. `CommitDetail` 组件
5. 复用 `DiffViewer` 展示 commit diff

**注意**：提交历史和 diff 的 `commitSha` 要作为参数传到 `get_diff`。

### Step 2-3：后台刷新

1. 实现 `services/watcher.rs`（notify crate 监听 `.git` 目录）
2. 在 `open_repository` 时启动监听
3. 在 `close_repository` 时停止监听
4. 前端 `useRepoEvents` hook 监听事件

**注意**：debounce 500ms，测试在 `git commit` 后 UI 是否自动刷新。

### Step 2-4：基础 Toast 系统 + 错误处理

确保所有错误（stage 失败、commit 失败、分支切换冲突）都有用户可读的提示。

---

## 5. 哪些部分先做 Stub/Mock

| 模块 | Stub 方案 |
|---|---|
| GitHub OAuth | `auth_service.rs` 返回 `Err(AppError::AuthRequired)` |
| Push/Pull/Fetch | command 存在但返回 `Err(AppError::NotImplemented)` |
| 提交图谱 | 前端组件返回 `<ComingSoon />` 占位 |
| 语法高亮 | DiffViewer 先纯文本，后期加 shiki |
| 应用更新 | 跳过，第二阶段再加 |

---

## 6. 哪些接口先打通

**必须在 Phase 1.1 结束前打通**（缺少这些会阻塞后续开发）：
1. `open_repository` ← 整个应用的入口
2. `get_status` ← 所有 Git 操作的基础
3. `get_diff` ← 用户最频繁查看的数据
4. `stage_files` + `commit_changes` ← 核心价值

**可以延后打通**（不影响 Phase 1.1 主干）：
- `get_branches` ← Phase 1.2
- `get_commit_log` ← Phase 1.2
- 后台刷新 ← Phase 1.2 末尾

---

## 7. 哪些 UI 先搭骨架

**骨架优先级（Phase 0 末尾完成）**：

```
1. AppLayout（TitleBar + 内容区域 + StatusBar）
2. WelcomePage（最近仓库列表 + 打开按钮）
3. RepoLayout（三栏骨架，Panel 用 div 占位）
4. 全局 Toast 容器

骨架内容可以是：
- 灰色方块（bg-surface-2 rounded）
- "TODO: BranchList" 文字占位
- 正确的布局比例和间距
```

**UI 细节优化顺序**（功能跑通后再做）：
1. 文件状态徽章样式（颜色、图标）
2. Diff 行颜色（绿色/红色背景）
3. 提交历史时间格式化（"2 hours ago"）
4. 空状态 UI（"No changes" / "No commits"）
5. 加载骨架屏
6. 动效（最后加）

---

## 8. 如何避免过度耦合

### 8.1 Rust 端

- `commands/` 函数**不包含任何业务逻辑**，只做参数传递和错误映射
- `services/` 之间**不相互持有引用**，通过参数传递需要的数据
- `GitService` **不知道** HTTP 的存在；`GithubApiService` **不知道** git 的存在
- 测试 `GitService` 方法时不需要 Tauri 运行时（纯 Rust 可测试）

### 8.2 前端

- **组件不直接调用 `invoke()`**，必须通过 `services/` 层
- **组件不直接读写 Zustand store**，通过 hooks 访问
- **TanStack Query 的 key 结构**统一管理（建议单独一个 `queryKeys.ts` 文件）
- **`DiffViewer` 组件只接受 `FileDiff` prop**，不知道数据来源（历史/工作区）

### 8.3 类型安全

- 所有前后端接口类型**必须来自 `bindings.ts`**（specta 生成）
- **不要手写重复的类型定义**
- 每次修改 Rust domain models 后，**立即重新生成 `bindings.ts`**

---

## 9. 关键实现细节提醒

### git2 index 操作

```rust
// Stage 文件的正确姿势
pub async fn stage_files(&self, paths: &[String]) -> Result<(), AppError> {
    let repo = self.repo.lock().await;
    let mut index = repo.index()?;
    for path in paths {
        let path = std::path::Path::new(path);
        if path.exists() {
            index.add_path(path)?;     // 新文件/修改文件
        } else {
            index.remove_path(path)?;  // 已删除的文件
        }
    }
    index.write()?;  // 必须 write，否则不生效
    Ok(())
}
```

### TanStack Query 失效策略

```typescript
// 每次 mutation 后，统一失效整个仓库的相关缓存
const invalidateRepo = (repoPath: string) => {
  queryClient.invalidateQueries({ queryKey: ['repo', repoPath, 'status'] });
  // commit 后还需失效 commits
  queryClient.invalidateQueries({ queryKey: ['repo', repoPath, 'commits'] });
};
```

### 分页 revwalk（注意性能）

```rust
// git2 revwalk 不支持 OFFSET，需要手动 skip
pub async fn get_commit_log(&self, page: u32, limit: u32) -> Result<CommitLogResult, AppError> {
    let repo = self.repo.lock().await;
    let mut walk = repo.revwalk()?;
    walk.push_head()?;
    walk.set_sorting(git2::Sort::TIME)?;
    
    let skip = (page * limit) as usize;
    let commits: Vec<_> = walk
        .skip(skip)
        .take(limit as usize + 1)  // 多取 1 个判断 hasMore
        .filter_map(|oid| oid.ok())
        .filter_map(|oid| repo.find_commit(oid).ok())
        .collect();
    
    let has_more = commits.len() > limit as usize;
    let commits = commits.into_iter().take(limit as usize)
        .map(|c| build_commit_summary(&c))
        .collect();
    
    Ok(CommitLogResult { commits, has_more })
}
```

---

## 10. 开发自检清单（每个功能模块完成前过一遍）

- [ ] Rust 类型定义有 `Serialize + Deserialize + specta::Type`
- [ ] 错误路径有对应的 `AppError` 变体
- [ ] Command 函数只做参数传递，不含业务逻辑
- [ ] 前端 service 函数有正确的 TypeScript 类型（来自 bindings.ts）
- [ ] Mutation 成功后 invalidate 了正确的 Query
- [ ] 危险操作（discard、delete branch）有确认对话框
- [ ] 边界情况（detached HEAD、unborn repo）有友好提示
- [ ] 大文件/二进制文件有截断处理
- [ ] 事件监听器在组件卸载时注销
