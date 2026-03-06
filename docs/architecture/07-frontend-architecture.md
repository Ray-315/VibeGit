# G. 前端架构设计

## 1. 整体结构

```
React 应用树
└── App.tsx（QueryClientProvider + Router + ThemeProvider）
    ├── AppLayout（TitleBar + 全局 Toast）
    │   ├── WelcomePage（无仓库时）
    │   └── RepoLayout（三栏布局）
    │       ├── Sidebar（仓库/分支树）
    │       ├── MainPanel（Tabs: Workspace / History）
    │       │   ├── WorkspaceView
    │       │   │   ├── StagingPanel（左）
    │       │   │   └── DiffViewer（右）
    │       │   └── HistoryView
    │       │       ├── CommitList（左）
    │       │       └── CommitDetail（右）
    │       └── StatusBar（底部）
    └── SettingsPage
```

---

## 2. 布局系统

### 2.1 三栏可调整布局

主布局采用三栏结构，支持拖拽调整宽度：

```
┌──────────┬───────────────────────┬──────────────────────┐
│          │                       │                      │
│ Sidebar  │    Main Panel         │   Detail Panel       │
│ 240px    │    (flex-1)           │   300-600px          │
│ (固定)   │                       │   (可拖拽)           │
│          │                       │                      │
└──────────┴───────────────────────┴──────────────────────┘
│                    StatusBar (32px)                      │
└──────────────────────────────────────────────────────────┘
```

实现方式：CSS Grid + 自定义拖拽 `ResizablePanel` 组件（不用 react-resizable-panels，避免额外依赖；实现简单版本即可）

```tsx
// components/shared/ResizablePanel.tsx
export function ResizablePanel({ 
  children, 
  defaultWidth, 
  minWidth = 200, 
  maxWidth = 800,
  side = 'right' 
}: ResizablePanelProps) {
  const [width, setWidth] = useState(defaultWidth);
  // 拖拽逻辑：mousedown → mousemove → mouseup
  // 使用 useRef 避免重渲染
}
```

### 2.2 自定义标题栏

Tauri 设置 `decorations: false`，前端实现自定义标题栏：

```tsx
// layouts/TitleBar.tsx
export function TitleBar() {
  return (
    <div 
      data-tauri-drag-region  // Tauri 拖动区域标记
      className="h-10 flex items-center px-3 select-none"
    >
      <AppIcon />
      <span className="text-sm font-medium ml-2">VibeGit</span>
      <div className="flex-1" data-tauri-drag-region />
      {/* 窗口控制按钮：最小化/最大化/关闭 */}
      <WindowControls />
    </div>
  );
}
```

---

## 3. 状态管理方案

### 3.1 选型决策

| 方案 | 适合场景 | 本项目评估 |
|---|---|---|
| **TanStack Query** ✅ | 服务端数据（fetch/cache/sync） | 主要数据层：commits、status、branches |
| **Zustand** ✅ | 客户端 UI 状态 | 选中状态、面板开关、toast 队列 |
| Redux Toolkit | 大型应用复杂状态 | 过重，不需要 |
| Jotai | 原子化状态 | 可以，但 Zustand 更直观 |
| Context API | 简单全局状态 | 太简单，不适合 |

**推荐**：TanStack Query（服务端数据）+ Zustand（UI 状态），两者协作，职责清晰。

### 3.2 TanStack Query 配置

```typescript
// App.tsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 3000,          // 3 秒内不重新获取
      gcTime: 5 * 60 * 1000,   // 5 分钟后清除缓存
      retry: (failureCount, error) => {
        // 不重试用户错误（如 NOT_A_REPO）
        if (isUserError(error)) return false;
        return failureCount < 2;
      },
      refetchOnWindowFocus: false, // 桌面应用不需要
    },
  },
});
```

Query Key 规范：
```typescript
// 分层结构便于精确失效
['repo', repoPath]                              // 仓库根
['repo', repoPath, 'summary']                   // 仓库摘要
['repo', repoPath, 'status']                    // 工作区状态
['repo', repoPath, 'branches']                  // 分支列表
['repo', repoPath, 'commits', { page, limit }]  // 提交历史
['repo', repoPath, 'commit', sha]               // 单个提交
['repo', repoPath, 'diff', { sha, path, context }]  // diff
```

### 3.3 Zustand Store 设计

```typescript
// store/repoStore.ts
interface RepoState {
  currentRepoPath: string | null;
  selectedCommitSha: string | null;
  selectedFilePath: string | null;
  activeView: 'workspace' | 'history';

  setCurrentRepo: (path: string | null) => void;
  setSelectedCommit: (sha: string | null) => void;
  setSelectedFile: (path: string | null) => void;
  setActiveView: (view: 'workspace' | 'history') => void;
}

// store/uiStore.ts
interface UIState {
  sidebarCollapsed: boolean;
  detailPanelWidth: number;
  setSidebarCollapsed: (v: boolean) => void;
  setDetailPanelWidth: (w: number) => void;
}

// store/toastStore.ts
interface ToastState {
  toasts: Toast[];
  addToast: (toast: Omit<Toast, 'id'>) => void;
  removeToast: (id: string) => void;
}
```

---

## 4. 命令调用封装

### 4.1 服务层封装

所有 `invoke` 调用封装在 `src/services/` 中，不直接在组件/hooks 中使用：

```typescript
// services/gitApi.ts
import { invoke } from '@tauri-apps/api/core';
import type { StatusSummary, AppError } from '../types/bindings';

export async function getStatus(repoPath: string): Promise<StatusSummary> {
  try {
    return await invoke<StatusSummary>('get_status', { repoPath });
  } catch (err) {
    throw parseApiError(err);
  }
}

// 解析 Rust 返回的错误 JSON
function parseApiError(err: unknown): ApiError {
  if (typeof err === 'string') {
    try {
      return JSON.parse(err) as ApiError;
    } catch {
      return { code: 'INTERNAL', message: String(err) };
    }
  }
  return { code: 'INTERNAL', message: 'Unknown error' };
}
```

### 4.2 Hooks 封装

```typescript
// hooks/useGitStatus.ts
export function useGitStatus(repoPath: string | null) {
  return useQuery({
    queryKey: ['repo', repoPath, 'status'],
    queryFn: () => getStatus(repoPath!),
    enabled: !!repoPath,
    refetchInterval: false, // 通过事件驱动刷新，不轮询
  });
}

// hooks/useStaging.ts
export function useStaging(repoPath: string) {
  const queryClient = useQueryClient();

  const stageFiles = useMutation({
    mutationFn: (paths: string[]) => stageFilesApi(repoPath, paths),
    onSuccess: () => {
      // Stage 成功后失效状态缓存
      queryClient.invalidateQueries({ queryKey: ['repo', repoPath, 'status'] });
    },
    onError: (error: ApiError) => {
      useToastStore.getState().addToast({
        type: 'error',
        title: 'Stage failed',
        message: error.message,
      });
    },
  });

  return { stageFiles };
}
```

---

## 5. 全局错误提示与 Loading 体系

### 5.1 Loading 策略

```
级别 1（全屏 Loading）：打开仓库、切换仓库
级别 2（局部骨架屏）：首次加载提交历史、分支列表
级别 3（内联 spinner）：stage/unstage/commit 操作
级别 4（按钮禁用态）：正在提交时禁用提交按钮
```

### 5.2 全局 Toast 系统

```tsx
// components/shared/Toaster.tsx
export function Toaster() {
  const { toasts, removeToast } = useToastStore();
  return (
    <div className="fixed bottom-4 right-4 z-50 flex flex-col gap-2">
      <AnimatePresence>
        {toasts.map((toast) => (
          <motion.div
            key={toast.id}
            initial={{ opacity: 0, y: 10, scale: 0.95 }}
            animate={{ opacity: 1, y: 0, scale: 1 }}
            exit={{ opacity: 0, y: -10, scale: 0.95 }}
            transition={{ duration: 0.15 }}
          >
            <ToastItem toast={toast} onDismiss={() => removeToast(toast.id)} />
          </motion.div>
        ))}
      </AnimatePresence>
    </div>
  );
}
```

### 5.3 错误边界

```tsx
// components/shared/ErrorBoundary.tsx
export class ErrorBoundary extends React.Component<Props, State> {
  // 捕获子树渲染错误，显示降级 UI
  // 记录错误到 tracing（通过 invoke('log_error')）
}
```

---

## 6. Diff 查看器设计

```tsx
// components/git/DiffViewer.tsx
export function DiffViewer({ diff }: { diff: FileDiff }) {
  if (diff.isBinary) return <BinaryDiffPlaceholder />;
  if (diff.hunks.length === 0) return <NoChangesPlaceholder />;

  return (
    <div className="font-mono text-xs leading-5">
      {diff.hunks.map((hunk, i) => (
        <DiffHunk key={i} hunk={hunk} />
      ))}
    </div>
  );
}

// components/git/DiffHunk.tsx
export function DiffHunk({ hunk }: { hunk: DiffHunk }) {
  return (
    <div>
      {/* @@ header */}
      <div className="text-muted-foreground bg-muted/30 px-4 py-0.5">
        {hunk.header}
      </div>
      {/* diff lines */}
      {hunk.lines.map((line, i) => (
        <DiffLineRow key={i} line={line} />
      ))}
    </div>
  );
}

function DiffLineRow({ line }: { line: DiffLine }) {
  const bg = {
    Addition: 'bg-emerald-500/10 text-emerald-300',
    Deletion: 'bg-red-500/10 text-red-300',
    Context:  'text-muted-foreground',
    HunkHeader: 'text-muted-foreground bg-muted/30',
  }[line.kind];

  return (
    <div className={`flex px-4 ${bg}`}>
      <span className="w-10 text-right mr-4 opacity-40 select-none">
        {line.oldLineno ?? ' '}
      </span>
      <span className="w-10 text-right mr-4 opacity-40 select-none">
        {line.newLineno ?? ' '}
      </span>
      <span className="flex-1 whitespace-pre-wrap break-all">
        {line.content}
      </span>
    </div>
  );
}
```

---

## 7. 动效设计原则

### 7.1 使用 Motion（Framer Motion 后继）

```typescript
// 动效配置常量
export const transitions = {
  fast: { duration: 0.1, ease: 'easeOut' },
  normal: { duration: 0.15, ease: 'easeOut' },
  smooth: { duration: 0.25, ease: [0.4, 0, 0.2, 1] },
};

// 常用动画变体
export const fadeIn = {
  initial: { opacity: 0 },
  animate: { opacity: 1 },
  exit: { opacity: 0 },
  transition: transitions.fast,
};

export const slideUp = {
  initial: { opacity: 0, y: 8 },
  animate: { opacity: 1, y: 0 },
  exit: { opacity: 0, y: -4 },
  transition: transitions.normal,
};
```

### 7.2 动效使用原则

| 场景 | 动效 | 时长 |
|---|---|---|
| Toast 出现/消失 | 淡入 + 上移 | 150ms |
| 面板切换 | 交叉淡入 | 150ms |
| 列表项出现 | 淡入（不 stagger） | 100ms |
| 模态框 | 淡入 + 缩放 | 150ms |
| 按钮点击 | 无动效（用颜色反馈） | — |
| 加载骨架屏 | shimmer（CSS animation） | 持续 |
| 进度条 | smooth width 过渡 | 200ms |

**禁止**：弹性动画（spring with high stiffness）、复杂多阶段入场动画、超过 300ms 的过渡

---

## 8. 暗色主题视觉系统

### 8.1 CSS 变量设计

```css
/* styles/themes/dark.css */
:root {
  /* 背景层级 */
  --background:    220 13% 9%;    /* 最深背景 #141618 */
  --surface-1:     220 13% 11%;   /* 面板背景 #181a1e */
  --surface-2:     220 13% 14%;   /* 悬浮/选中背景 #1e2126 */
  --surface-3:     220 13% 18%;   /* 输入框背景 #262b33 */

  /* 前景层级 */
  --foreground:    220 15% 92%;   /* 主文字 #e8eaed */
  --muted-foreground: 220 10% 55%; /* 次要文字 #818999 */
  --placeholder:   220 10% 35%;   /* 占位文字 */

  /* 强调色 */
  --primary:       217 91% 60%;   /* 蓝色 #3b82f6 */
  --primary-hover: 217 91% 65%;
  --success:       142 71% 45%;   /* 绿色（新增）*/
  --danger:        0 84% 60%;     /* 红色（删除）*/
  --warning:       38 92% 50%;    /* 黄色（冲突）*/

  /* 边框 */
  --border:        220 13% 18%;   /* 默认边框 */
  --border-strong: 220 13% 25%;   /* 强调边框 */

  /* 圆角 */
  --radius: 6px;
  --radius-sm: 4px;
  --radius-lg: 8px;

  /* 阴影 */
  --shadow-sm: 0 1px 3px rgba(0,0,0,0.3);
  --shadow-md: 0 4px 12px rgba(0,0,0,0.4);
  --shadow-lg: 0 8px 24px rgba(0,0,0,0.5);
}
```

### 8.2 字体选择

```css
/* 界面字体：Inter Variable（清晰、现代、免费） */
font-family: 'Inter Variable', system-ui, -apple-system, sans-serif;

/* 代码/diff 字体：JetBrains Mono（开发者熟悉，支持 ligatures） */
font-family: 'JetBrains Mono', 'Cascadia Code', 'Consolas', monospace;
```

字体使用 CSS `@font-face` 本地加载，不依赖网络。打包时内嵌到应用中。

### 8.3 视觉层级

```
层级 0 — 窗口背景（--background）
层级 1 — 面板背景（--surface-1）：Sidebar、MainPanel
层级 2 — 卡片/选中项（--surface-2）
层级 3 — 输入框/代码块（--surface-3）
弹出层  — 下拉菜单、模态框（surface-1 + shadow-lg）
```

---

## 9. 后台事件监听

```typescript
// hooks/useRepoEvents.ts
export function useRepoEvents(repoPath: string | null) {
  const queryClient = useQueryClient();

  useEffect(() => {
    if (!repoPath) return;

    // 监听仓库文件变化
    const unlisten = listen('repo:changed', (event) => {
      if (event.payload.repoPath === repoPath) {
        queryClient.invalidateQueries({ queryKey: ['repo', repoPath, 'status'] });
        queryClient.invalidateQueries({ queryKey: ['repo', repoPath, 'summary'] });
      }
    });

    return () => {
      unlisten.then((fn) => fn());
    };
  }, [repoPath, queryClient]);
}
```
