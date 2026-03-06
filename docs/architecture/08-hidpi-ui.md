# H. 高分屏与 UI 适配设计

## 1. 核心问题分析

Windows 高分辨率屏幕适配涉及以下层次：

```
物理像素（Physical Pixels）
    ↑ DPI Scaling（操作系统层，Windows 100%/125%/150%/200%）
逻辑像素（Logical Pixels / CSS Pixels）
    ↑ WebView 渲染（WebView2 / Edge）
CSS 布局单位（rem / px / em）
    ↑ 前端代码
```

---

## 2. Tauri + WebView2 的 HiDPI 表现

### 2.1 基本原理

- **WebView2** 在 Windows 上使用 **DPI-Aware 模式**，自动处理 DPI 缩放
- CSS `1px` = 1 个**逻辑像素**（CSS Pixel）
- Windows 200% DPI 下：1 个 CSS Pixel = 2 × 2 个物理像素
- 文字、矢量图形（SVG）、CSS 绘制的边框/阴影 → **自动清晰**
- 位图图片（PNG/JPEG）→ 需要提供 2x/3x 版本或改用 SVG

### 2.2 Tauri 窗口尺寸 API

```rust
// Tauri 2 提供逻辑/物理尺寸两套 API
use tauri::LogicalSize;

window.set_min_size(Some(LogicalSize::new(900.0, 600.0)))?;
// LogicalSize 会自动换算为物理像素
```

### 2.3 不会模糊的根本原因

Tauri + WebView2 的 UI 渲染路径：
```
React 渲染 → CSS 布局 → WebView2（DirectWrite/Skia）→ GPU 光栅化
```
- 文字由 DirectWrite 渲染，天然支持 ClearType 和亚像素抗锯齿
- CSS 边框、阴影、圆角由 Skia 在物理像素层面计算
- 只要不使用位图缩放，就不会出现模糊

---

## 3. 常见 DPI 场景下的布局原则

### 3.1 缩放比例对照

| Windows 缩放 | DPI | devicePixelRatio | 建议字体大小（逻辑px） |
|---|---|---|---|
| 100% | 96 DPI | 1 | 13px |
| 125% | 120 DPI | 1.25 | 13px |
| 150% | 144 DPI | 1.5 | 13px |
| 200% | 192 DPI | 2 | 13px |

> **关键结论**：前端**不需要**根据 DPI 修改 CSS 大小——WebView2 已经处理好缩放，保持 CSS 值不变即可。

### 3.2 字体大小建议

```css
/* 全局基准字体大小 */
html { font-size: 14px; }

/* 界面文字层级 */
.text-xs   { font-size: 11px; }   /* 辅助说明、时间戳 */
.text-sm   { font-size: 12px; }   /* 次要信息 */
.text-base { font-size: 13px; }   /* 正文（主要内容）*/
.text-lg   { font-size: 15px; }   /* 标题、重要信息 */

/* Diff / 代码区字体 */
.font-mono { font-size: 12px; line-height: 18px; }
```

> **避免低于 11px**：高分屏物理像素充足，但过小字体在用户调整系统字体时会失控。

---

## 4. 图标策略

### 4.1 使用 SVG 图标（推荐）

所有 UI 图标使用 SVG 格式，推荐 **Lucide React**（已内置）：

```tsx
import { GitBranch, Check, X, Plus } from 'lucide-react';

// 始终使用逻辑像素指定大小
<GitBranch size={14} />   // 分支图标
<Check size={16} />        // 确认图标
```

优点：
- SVG 在任何 DPI 下完美清晰
- 体积小，可主题化
- 不需要 2x/3x 多套资源

### 4.2 应用图标（桌面 + 任务栏）

Tauri 打包需要提供多尺寸 PNG：
```
icons/
  32x32.png
  128x128.png
  128x128@2x.png
  icon.ico        # Windows（包含 16/32/48/256 多尺寸）
  icon.icns       # macOS
```

---

## 5. 边框、阴影、分割线策略

### 5.1 边框

```css
/* 使用 CSS border（不用 outline 或 box-shadow 模拟）*/
.panel-border {
  border: 1px solid hsl(var(--border));
}

/* 分割线使用 1px border，不会在高分屏上变粗 */
.divider {
  border-top: 1px solid hsl(var(--border));
}
```

> **不使用 `0.5px` 边框技巧**：WebView2 会正确将 1px CSS 边框渲染为 1 个逻辑像素（在 200% DPI 下 = 2 物理像素），视觉效果是恰好 1 物理像素的细线。无需额外处理。

### 5.2 阴影

```css
/* 使用较小的模糊半径，避免在高分屏显得过重 */
.shadow-panel {
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.3);
}

.shadow-popup {
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.4);
}
```

### 5.3 圆角

```css
/* 统一圆角值，高分屏下圆角会更平滑 */
--radius: 6px;
--radius-sm: 4px;
--radius-lg: 10px;
```

---

## 6. 复杂区域适配建议

### 6.1 Diff 查看器

- 使用 `font-mono` 等宽字体（JetBrains Mono）
- 行高设为 `18px`（逻辑像素），确保高分屏下行距舒适
- 横向滚动：`overflow-x: auto; white-space: pre;`（不换行）
- 行号列宽固定（`width: 40px`），内容列 `flex-1`

```tsx
<div className="overflow-x-auto">
  <div className="min-w-0 font-mono text-xs leading-[18px]">
    {/* diff 内容 */}
  </div>
</div>
```

### 6.2 提交历史列表

- 使用虚拟列表（自定义或 `@tanstack/react-virtual`）
- 每行高度固定 `56px`（逻辑像素）
- 避免 re-render：每行为纯展示组件，`React.memo` 包装

```tsx
const CommitRow = React.memo(({ commit }: { commit: CommitSummary }) => (
  <div className="h-14 flex items-center px-4 gap-3 hover:bg-surface-2 cursor-pointer">
    {/* 提交信息 */}
  </div>
));
```

### 6.3 文件树

- 树节点缩进：每层 `16px`（不是 `1rem`，避免字体缩放影响）
- 图标 14px，文字 12px
- 选中态：`bg-surface-2` + 左侧 `2px` 彩色边框

---

## 7. 窗口缩放与最小尺寸

### 7.1 最小窗口尺寸设置

```json
// tauri.conf.json
{
  "windows": [{
    "minWidth": 900,
    "minHeight": 600
  }]
}
```

### 7.2 响应式布局断点

虽然是桌面应用，仍需考虑用户缩小窗口的场景：

```
≥ 1400px：三栏全展开，Sidebar 240px，Detail 400px
1200–1400px：三栏，Sidebar 220px，Detail 340px
900–1200px：侧边栏可折叠，Detail 面板可收起为 icon
< 900px：不允许（minWidth 限制）
```

实现：使用 `ResizeObserver` 监听容器宽度，动态添加 CSS class：

```tsx
// hooks/useContainerSize.ts
export function useContainerSize(ref: RefObject<HTMLElement>) {
  const [size, setSize] = useState({ width: 0, height: 0 });
  
  useEffect(() => {
    if (!ref.current) return;
    const observer = new ResizeObserver(([entry]) => {
      setSize({
        width: entry.contentRect.width,
        height: entry.contentRect.height,
      });
    });
    observer.observe(ref.current);
    return () => observer.disconnect();
  }, [ref]);

  return size;
}
```

---

## 8. 桌面原生感实现策略

### 8.1 "网页感"的根本原因

| 网页感问题 | 解决方案 |
|---|---|
| 文字可选中、可拖拽 | `user-select: none`（除 diff/内容区） |
| 右键默认浏览器菜单 | `onContextMenu` 拦截，弹出自定义菜单 |
| 蓝色链接下划线 | 全局 reset，链接用 hover 颜色区分 |
| 滚动条样式丑 | CSS 自定义滚动条（细滚动条，暗色主题） |
| 拖拽选中文字 | `draggable="false"` + CSS |
| 点击有 outline | `outline: none` + focus-visible 样式 |
| 过度弹跳感 | 减少 spring animation，使用 easeOut |
| 字体渲染模糊 | 使用变量字体（Inter Variable），避免非系统字体 |

### 8.2 自定义滚动条

```css
/* 细滚动条，符合桌面工具视觉 */
::-webkit-scrollbar { width: 6px; height: 6px; }
::-webkit-scrollbar-track { background: transparent; }
::-webkit-scrollbar-thumb {
  background: hsl(var(--border-strong));
  border-radius: 3px;
}
::-webkit-scrollbar-thumb:hover {
  background: hsl(var(--muted-foreground));
}
```

### 8.3 全局禁止文字选中

```css
body {
  -webkit-user-select: none;
  user-select: none;
}

/* 允许内容区选中 */
.selectable {
  -webkit-user-select: text;
  user-select: text;
}
```

---

## 9. 字体加载策略

使用 CSS `@font-face` 从应用内嵌资源加载，**不依赖网络**：

```css
/* 使用 Tauri protocol-asset 提供字体文件 */
@font-face {
  font-family: 'Inter Variable';
  src: url('/assets/fonts/InterVariable.woff2') format('woff2-variations');
  font-weight: 100 900;
  font-display: block; /* 确保字体加载前不显示内容 */
}

@font-face {
  font-family: 'JetBrains Mono';
  src: url('/assets/fonts/JetBrainsMono-Regular.woff2') format('woff2');
  font-weight: 400;
  font-display: block;
}
```

字体文件放在 `src-tauri/assets/fonts/`，通过 Tauri 的 `asset:` 协议提供。
