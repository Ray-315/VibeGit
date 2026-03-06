# D. 目录结构设计

## 1. 完整目录树

```
VibeGit/
├── .github/
│   └── workflows/
│       ├── ci.yml              # 持续集成（lint + build + test）
│       └── release.yml         # 发布构建（tauri build）
│
├── docs/
│   └── architecture/           # 架构设计文档（本目录）
│       ├── 00-overview.md
│       ├── 01-tech-selection.md
│       └── ...
│
├── src-tauri/                  # Rust / Tauri 后端
│   ├── Cargo.toml
│   ├── Cargo.lock
│   ├── build.rs                # Tauri 构建脚本
│   ├── tauri.conf.json         # Tauri 配置（权限、窗口、bundle）
│   ├── capabilities/           # Tauri 2 细粒度权限配置
│   │   └── default.json
│   ├── icons/                  # 应用图标（多尺寸 PNG + ICO）
│   └── src/
│       ├── main.rs             # 入口：仅启动 Tauri app
│       ├── lib.rs              # 插件注册 + command 注册 + AppState 初始化
│       ├── state.rs            # AppState 结构体定义
│       │
│       ├── commands/           # Tauri command 层（胶水层）
│       │   ├── mod.rs          # 统一导出所有 command
│       │   ├── repo.rs         # 仓库管理命令（open/close/recent）
│       │   ├── git.rs          # Git 操作命令（status/stage/commit 等）
│       │   ├── branch.rs       # 分支管理命令
│       │   ├── history.rs      # 提交历史命令
│       │   ├── diff.rs         # Diff 查看命令
│       │   └── auth.rs         # GitHub 认证命令（第二阶段）
│       │
│       ├── services/           # 业务逻辑服务层
│       │   ├── mod.rs
│       │   ├── git_service.rs  # 封装 git2，核心 Git 操作
│       │   ├── repo_manager.rs # 仓库注册表 + 最近列表管理
│       │   ├── watcher.rs      # 文件系统变化监听（notify crate）
│       │   ├── auth_service.rs # GitHub OAuth + keyring 凭证管理
│       │   └── github_api.rs   # GitHub REST API 封装（第二阶段）
│       │
│       ├── domain/             # 领域模型与类型
│       │   ├── mod.rs
│       │   ├── models.rs       # 所有领域数据结构（serde + specta）
│       │   └── error.rs        # AppError 枚举（thiserror）
│       │
│       └── utils/              # 工具函数
│           ├── mod.rs
│           ├── path.rs         # 路径规范化（跨平台处理）
│           └── encoding.rs     # 文件编码检测
│
├── src/                        # 前端 React + TypeScript
│   ├── main.tsx                # React 入口
│   ├── App.tsx                 # 根组件 + 路由 + 全局 Provider
│   │
│   ├── pages/                  # 路由级页面组件
│   │   ├── Welcome.tsx         # 欢迎页（无仓库时）
│   │   ├── Repository.tsx      # 主工作区页面（有仓库时）
│   │   └── Settings.tsx        # 设置页面
│   │
│   ├── layouts/                # 布局组件
│   │   ├── AppLayout.tsx       # 整体窗口布局
│   │   ├── RepoLayout.tsx      # 三栏布局（侧边栏 + 内容 + 详情）
│   │   └── TitleBar.tsx        # 自定义标题栏（含窗口控制按钮）
│   │
│   ├── components/             # 可复用 UI 组件
│   │   ├── ui/                 # shadcn/ui 基础组件（本地拷贝，不黑盒）
│   │   │   ├── button.tsx
│   │   │   ├── dialog.tsx
│   │   │   ├── dropdown-menu.tsx
│   │   │   ├── input.tsx
│   │   │   ├── scroll-area.tsx
│   │   │   ├── separator.tsx
│   │   │   ├── skeleton.tsx
│   │   │   ├── tabs.tsx
│   │   │   ├── toast.tsx
│   │   │   └── tooltip.tsx
│   │   │
│   │   ├── git/                # Git 领域专用组件
│   │   │   ├── BranchSelector.tsx       # 分支切换下拉
│   │   │   ├── CommitList.tsx           # 提交历史列表（虚拟列表）
│   │   │   ├── CommitDetail.tsx         # 提交详情面板
│   │   │   ├── FileChangeList.tsx       # 变更文件列表
│   │   │   ├── FileStatusBadge.tsx      # 文件状态徽章（M/A/D/R）
│   │   │   ├── DiffViewer.tsx           # Diff 查看器（统一视图）
│   │   │   ├── DiffHunk.tsx             # 单个 diff hunk 渲染
│   │   │   ├── CommitMessageEditor.tsx  # Commit 消息输入框
│   │   │   ├── StagingPanel.tsx         # 暂存区面板（左侧）
│   │   │   └── StatusBar.tsx            # 底部状态栏
│   │   │
│   │   └── shared/             # 通用工具组件
│   │       ├── EmptyState.tsx  # 空状态占位图
│   │       ├── ErrorBoundary.tsx
│   │       ├── LoadingSpinner.tsx
│   │       ├── ResizablePanel.tsx    # 可拖拽调整宽度面板
│   │       └── VirtualList.tsx       # 虚拟列表容器
│   │
│   ├── hooks/                  # 自定义 React Hooks
│   │   ├── useRepo.ts          # 仓库状态管理 hook
│   │   ├── useGitStatus.ts     # 工作区状态查询
│   │   ├── useBranches.ts      # 分支列表查询
│   │   ├── useCommitLog.ts     # 提交历史查询（分页）
│   │   ├── useCommitDetail.ts  # 单个 commit 详情
│   │   ├── useDiff.ts          # diff 数据查询
│   │   ├── useStaging.ts       # stage / unstage 操作
│   │   ├── useCommit.ts        # 提交操作
│   │   ├── useRepoEvents.ts    # 监听 Rust 后台事件
│   │   ├── useKeyboard.ts      # 键盘快捷键注册
│   │   └── useToast.ts         # Toast 通知 hook
│   │
│   ├── store/                  # Zustand 状态 store
│   │   ├── index.ts            # 统一导出
│   │   ├── repoStore.ts        # 当前仓库路径、选中状态
│   │   ├── uiStore.ts          # UI 状态（面板折叠、活跃视图）
│   │   └── toastStore.ts       # 全局 Toast 队列
│   │
│   ├── services/               # Tauri invoke 封装层
│   │   ├── index.ts            # 统一导出
│   │   ├── repoApi.ts          # 仓库相关 API
│   │   ├── gitApi.ts           # Git 操作 API
│   │   ├── branchApi.ts        # 分支操作 API
│   │   ├── historyApi.ts       # 历史查询 API
│   │   ├── diffApi.ts          # Diff 查询 API
│   │   └── authApi.ts          # 认证 API（第二阶段）
│   │
│   ├── types/                  # TypeScript 类型定义
│   │   ├── index.ts            # 统一导出
│   │   ├── bindings.ts         # specta 自动生成的 Rust 类型映射
│   │   ├── api.ts              # API 请求/响应类型
│   │   └── ui.ts               # 纯 UI 状态类型
│   │
│   ├── lib/                    # 工具函数库
│   │   ├── cn.ts               # clsx + tailwind-merge 工具
│   │   ├── format.ts           # 时间/大小格式化
│   │   ├── diff.ts             # diff 数据处理工具
│   │   └── constants.ts        # 全局常量
│   │
│   └── styles/                 # 全局样式
│       ├── globals.css         # Tailwind 基础层 + CSS 变量
│       └── themes/
│           ├── dark.css        # 暗色主题 CSS 变量
│           └── light.css       # 亮色主题 CSS 变量
│
├── index.html                  # Vite 入口 HTML
├── package.json
├── pnpm-lock.yaml
├── vite.config.ts
├── tsconfig.json
├── tailwind.config.ts
├── postcss.config.js
├── .eslintrc.json
├── .prettierrc
└── README.md
```

---

## 2. 各目录职责说明

### `src-tauri/src/commands/`
- **职责**：Tauri IPC 边界层，只做参数验证 + service 调用 + 错误映射
- **规则**：每个函数不超过 20 行，不含业务逻辑
- **文件拆分原则**：按功能域拆分，每个文件对应一个功能域

### `src-tauri/src/services/`
- **职责**：核心业务逻辑，封装 git2、HTTP 请求、文件系统操作
- **规则**：服务之间通过函数参数传递，不相互持有引用，避免循环依赖
- **GitService**：最复杂的服务，每个 git 操作封装为独立的方法

### `src-tauri/src/domain/`
- **职责**：纯数据定义，无业务逻辑，无 I/O 依赖
- **规则**：所有模型必须 `#[derive(Serialize, Deserialize, specta::Type)]`

### `src/services/`
- **职责**：封装所有 `invoke()` 调用，提供类型安全的 async 函数
- **规则**：每个函数必须有明确的输入/输出 TypeScript 类型，统一处理 invoke 错误

### `src/hooks/`
- **职责**：将 TanStack Query + Zustand + services 组合为 React 友好的接口
- **规则**：组件不直接调用 `invoke()`，只通过 hooks 获取数据和触发操作

### `src/store/`
- **职责**：管理纯 UI 状态（不缓存从 Rust 获取的数据）
- **规则**：只存放"前端决策需要的状态"，服务端数据交给 TanStack Query

### `src/types/bindings.ts`
- **职责**：与 Rust domain/models.rs 保持同步的 TypeScript 类型
- **规则**：通过 `specta` + `tauri-specta` 自动生成，不手写

---

## 3. 配置文件说明

### `tauri.conf.json` 关键配置
```json
{
  "app": {
    "windows": [{
      "title": "VibeGit",
      "width": 1280,
      "height": 800,
      "minWidth": 900,
      "minHeight": 600,
      "decorations": false,
      "transparent": false
    }]
  },
  "bundle": {
    "identifier": "com.vibegit.app",
    "icon": ["icons/32x32.png", "icons/icon.ico"]
  }
}
```

### `tailwind.config.ts` 关键配置
```typescript
export default {
  content: ['./index.html', './src/**/*.{ts,tsx}'],
  darkMode: 'class',
  theme: {
    extend: {
      colors: {
        // 从 CSS 变量读取，支持主题切换
        background: 'hsl(var(--background))',
        foreground: 'hsl(var(--foreground))',
        // ...
      },
      fontFamily: {
        sans: ['Inter Variable', 'system-ui', 'sans-serif'],
        mono: ['JetBrains Mono', 'Cascadia Code', 'monospace'],
      },
    },
  },
}
```
