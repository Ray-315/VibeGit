# L. 开发路线图

## 第 0 阶段：初始化工程

**目标**：搭建可运行的空壳，验证技术路线可行，建立工程规范。

**交付物**：
- Tauri 2 + React + TypeScript + Tailwind CSS 工程可启动（`pnpm tauri dev`）
- 自定义标题栏（draggable + 窗口控制按钮）
- 暗色主题 CSS 变量系统就绪
- shadcn/ui 基础组件集成（Button、Dialog、Toast、Input）
- Rust 端 AppState + 基础错误类型定义
- specta + tauri-specta 类型生成配置就绪
- 前端目录结构搭建完毕（pages/components/hooks/store/services/types）
- ESLint + Prettier + cargo fmt 代码规范配置
- CI 工作流（GitHub Actions：lint + build 验证）

**技术重点**：
- 验证 Windows HiDPI 渲染效果
- 验证 Tauri invoke 基础通信
- 验证 specta 类型生成流程

**风险点**：
- WebView2 版本兼容性（建议用最新 Tauri 2 稳定版）
- pnpm + Cargo 双包管理器工作流熟悉度

**建议测试项**：
- 手动测试：在 100% / 150% / 200% DPI 下窗口显示正常
- 手动测试：invoke 一个 hello world command，前后端通信正常
- CI 检查：`pnpm lint && pnpm build` 通过

---

## 第 1 阶段：MVP

**目标**：完成所有核心 Git 日常操作，能够替代基础 GUI 工具（如 GitKraken 的 20% 功能）。

### 第 1.1 阶段（核心工作流）

**交付物**：
- 仓库打开 / 最近仓库列表
- 工作区状态读取（已修改/已暂存/未跟踪文件列表）
- Diff 查看（文本文件 unified diff，含行号）
- Stage / Unstage 单文件和全部文件
- Discard Changes（带确认对话框）
- 撰写 commit 消息并提交
- 全局 Toast 错误提示

**技术重点**：
- git2 status 正确处理各种文件状态
- diff 大文件截断策略
- 暂存区 UI 双栏布局（已暂存/未暂存）

**风险点**：
- git2 stage/unstage 的 index 操作容易出错（需覆盖 renamed/deleted 等边界 case）
- 大仓库 status 性能（建议提前测试真实大仓库）

**建议测试项**：
- 测试 10+ 种文件状态（新建、修改、删除、重命名、复制）
- 手动测试 stage → commit 完整流程
- 测试 discard changes 不影响未选中文件

---

### 第 1.2 阶段（补全 MVP）

**交付物**：
- 分支列表（本地 + 远程跟踪分支）
- 分支切换（checkout）
- 创建/删除本地分支
- 提交历史列表（虚拟列表，可无限滚动）
- 点击 commit 查看详情（文件列表 + diff）
- 文件系统后台监听（repo:changed 事件）
- 基础设置页（用户名/邮箱）

**技术重点**：
- 虚拟列表性能（TanStack Virtual）
- 分支切换时工作区冲突处理
- 后台文件监听不引起频繁刷新（debounce）

**风险点**：
- git2 revwalk 分页实现（不天然支持 skip，需自行计数）
- 文件监听在 Windows 网络驱动器上不可靠（文档说明即可）

**建议测试项**：
- 滚动 500+ 条提交历史性能测试
- 测试分支切换工作区有修改的情况
- 测试在另一个终端 commit 后，UI 自动刷新

---

## 第 2 阶段：增强版

**目标**：提升用户体验和功能覆盖，接近主流 Git GUI 工具（如 Tower/Fork 的 50% 功能）。

**交付物**：
- 提交历史搜索 / 按分支过滤
- 并排 Diff 视图（Side-by-side）
- Diff 语法高亮（基于 shiki，离线）
- Stash 管理（Stash / Pop / Drop / List）
- Tag 列表 / 创建 / 删除
- 提交图谱基础版（分支拓扑 DAG，SVG 渲染）
- GitHub OAuth 登录（Device Flow）
- Push / Pull / Fetch 操作 + 进度条
- 克隆 GitHub 仓库
- 应用自动更新（tauri-plugin-updater）
- 亮色主题支持

**技术重点**：
- shiki 离线语法高亮集成（WebAssembly 版本）
- 提交图谱 DAG 算法（拓扑排序 + 列分配算法）
- GitHub Device Flow OAuth 实现
- HTTPS 凭证管理（keyring + Windows Credential Manager）

**风险点**：
- DAG 渲染对大仓库性能压力大（需要虚拟化）
- GitHub API 速率限制（需实现 token 缓存和 rate limit 处理）
- Push/Pull 进度回调在 Windows 上的 SSH 代理集成

---

## 第 3 阶段：进阶能力

**目标**：对标专业级 Git GUI 工具，覆盖高级 Git 操作和团队协作场景。

**交付物**：
- Merge UI（选择合并策略、快进提示）
- Rebase 操作（基础版：rebase onto 分支）
- 冲突解决视图（三路合并，文件级选择）
- Interactive Rebase（拖拽排序、squash、fixup）
- Blame 视图（行级提交追踪）
- GitHub PR 管理（列表 / 创建 / 审查评论）
- GitHub Actions 状态查看
- Worktree 管理
- Submodule 基础支持
- 命令面板（Command Palette，Ctrl+K）
- 多仓库标签页
- 本地 Git Hooks 管理

**技术重点**：
- 冲突解决三路合并 UI（复杂前端组件）
- Interactive rebase 的 todo 文件 UI 编辑
- PR 审查评论的实时更新（WebSocket / 轮询）

**风险点**：
- 冲突解决 UI 是最复杂的功能，需要专项设计
- Interactive rebase 需要精确的 git2 API 支持

---

## 里程碑时间参考（供参考，按团队实际调整）

| 阶段 | 建议工期 | 关键验证点 |
|---|---|---|
| Phase 0 | 1 周 | 工程跑起来，HiDPI 渲染正常 |
| Phase 1.1 | 2–3 周 | 能完成 stage → commit 完整流程 |
| Phase 1.2 | 2–3 周 | 能完整替代 `git log --oneline` + 分支操作 |
| Phase 2 | 4–6 周 | GitHub 集成 + 提交图谱可用 |
| Phase 3 | 持续迭代 | 按用户反馈优先级排序 |
