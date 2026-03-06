# A. 技术路线评估与最终选型

## 1. 方案对比矩阵

| 维度 | Tauri 2 + Rust + React | Electron + Node.js + React | 纯 Rust GUI（Slint / egui） |
|---|---|---|---|
| **包体积** | ✅ ~8–15 MB | ❌ ~120–250 MB（含 Chromium） | ✅ ~5–10 MB |
| **内存占用** | ✅ ~30–60 MB | ❌ ~150–300 MB | ✅ ~10–30 MB |
| **UI 设计自由度** | ✅ 全 Web 自由度 | ✅ 全 Web 自由度 | ⚠️ 受限于 Rust GUI 生态 |
| **高分屏适配** | ✅ WebView 原生 HiDPI | ✅ Chromium 原生 HiDPI | ⚠️ 需手动处理 DPI |
| **原生能力调用** | ✅ Rust IPC，无 Node.js 中间层 | ⚠️ Node.js 中间层，性能稍差 | ✅ 直接系统调用 |
| **Git 能力** | ✅ git2/libgit2（Rust） | ⚠️ nodegit / CLI 调用 | ✅ git2（Rust） |
| **开发效率** | ✅ React 生态成熟 | ✅ React 生态成熟 | ❌ Rust UI 生态不成熟 |
| **长期维护** | ✅ Tauri 社区活跃 | ⚠️ 体积/内存压力大 | ❌ UI 生态碎片化 |
| **跨平台能力** | ✅ Win/macOS/Linux | ✅ Win/macOS/Linux | ✅ Win/macOS/Linux |
| **安全沙箱** | ✅ 严格权限控制 | ⚠️ 更开放，需手动收窄 | ✅ 无 WebView |
| **动效/视觉** | ✅ CSS/Motion 全支持 | ✅ CSS/Motion 全支持 | ❌ 动效能力弱 |

---

## 2. 方案逐项分析

### 2.1 Tauri 2 + Rust + React（推荐方案）

**性能**
- Rust 后端处理所有 Git 操作，无 GIL、无 GC 停顿，大仓库遍历/diff 计算比 Node.js 快 2–5×
- WebView 渲染层由操作系统提供（Windows: WebView2 / Edge, macOS: WKWebView），不捆绑完整浏览器
- Tauri 2 的 IPC 为结构化消息传递，序列化开销小于 Electron 的 contextBridge

**包体积**
- 最终安装包 8–15 MB（不含 OS 级 WebView，Windows 10+ 内置 WebView2）
- 对比 Electron：省去 ~100 MB 的 Chromium 运行时

**UI 设计自由度**
- 使用 React + Tailwind CSS + shadcn/ui，完全继承 Web 设计生态
- 支持 CSS 动画、Framer Motion / Motion、SVG 图标、WebGL（未来扩展提交图谱）
- 任何现代 Web UI 效果均可实现

**高分屏适配**
- WebView2 原生支持 DPI 感知，HTML/CSS 逻辑像素体系天然适配
- Tauri 2 提供 `logical_size` / `physical_size` API，可精确控制窗口尺寸
- 字体渲染由 WebView2 负责，ClearType/DirectWrite 抗锯齿支持良好

**原生能力调用**
- Tauri commands 直接在 Rust 进程中执行，无需经过 Node.js
- 文件系统、系统托盘、全局快捷键、原生对话框均有 Tauri plugin 支持
- keyring（系统凭证存储）通过 `keyring` crate 直接调用 Windows Credential Manager

**开发效率**
- 前端开发者使用熟悉的 React/TypeScript 工具链（Vite、ESLint、Prettier）
- Rust 端热重载：`cargo-watch` + Tauri dev 模式
- 前后端接口通过 TypeScript 类型定义文件（`bindings/`）共享，可用 `ts-rs` / `specta` 自动生成

**长期维护成本**
- Tauri 2 由 Tauri Apps 组织维护，已发布稳定版，社区活跃
- git2 crate 封装成熟 libgit2，已被 GitButler 等产品级工具采用
- 前端依赖 shadcn/ui 等组件，维护成本低（本地拷贝，无黑盒依赖）

---

### 2.2 Electron（备选方案，不推荐）

**优点**
- 生态最成熟，参考项目多（VS Code、GitHub Desktop）
- Node.js 原生模块生态丰富

**缺点**
- 包体积 100 MB+ 严重影响分发体验
- 内存占用高（每个窗口独立 Chromium 进程）
- Git 能力依赖 nodegit（维护不活跃）或命令行 shelling out（性能差、解析脆弱）
- 对于本项目定位（高颜值桌面工具），Electron 的"网页感"更难克服

**结论**：不推荐。仅当团队完全不熟悉 Rust 时作为降级选项。

---

### 2.3 纯 Rust GUI — Slint / egui（不推荐）

**优点**
- 包体积最小，渲染性能极高
- 无 WebView 依赖，完全原生

**缺点**
- UI 设计自由度受限，难以实现"现代桌面效率工具"的视觉气质
- 动效、渐变、阴影、毛玻璃等效果实现成本极高
- 组件生态不成熟，设计系统几乎需从零构建
- 开发效率低，迭代速度慢

**结论**：不推荐。适合工具/系统软件，不适合本项目产品定位。

---

## 3. 最终选型结论

```
推荐方案：Tauri 2 + Rust + React + TypeScript
UI 层：    Tailwind CSS + shadcn/ui + Motion
Git 层：   git2 crate (libgit2 bindings)
状态层：   Zustand + TanStack Query
构建工具： Vite + pnpm
类型共享： specta + tauri-specta（自动生成 TS 绑定）
```

---

## 4. 核心依赖清单

### Rust 依赖（Cargo.toml）

```toml
[dependencies]
tauri          = { version = "2", features = ["protocol-asset"] }
tauri-plugin-dialog   = "2"
tauri-plugin-fs       = "2"
tauri-plugin-shell    = "2"
tauri-plugin-window-state = "2"

git2           = { version = "0.19", features = ["vendored-openssl"] }
serde          = { version = "1", features = ["derive"] }
serde_json     = "1"
thiserror      = "1"
anyhow         = "1"
tokio          = { version = "1", features = ["full"] }
reqwest        = { version = "0.12", features = ["json", "rustls-tls"] }
keyring        = "2"
specta         = { version = "2", features = ["derive"] }
tauri-specta   = { version = "2", features = ["derive"] }
tracing        = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
chrono         = { version = "0.4", features = ["serde"] }
```

### 前端依赖（package.json）

```json
{
  "dependencies": {
    "@tauri-apps/api": "^2",
    "@tauri-apps/plugin-dialog": "^2",
    "@tauri-apps/plugin-fs": "^2",
    "react": "^18",
    "react-dom": "^18",
    "zustand": "^4",
    "@tanstack/react-query": "^5",
    "motion": "^11",
    "tailwindcss": "^3",
    "@radix-ui/react-*": "latest",
    "lucide-react": "latest",
    "clsx": "latest",
    "tailwind-merge": "latest"
  },
  "devDependencies": {
    "vite": "^5",
    "@vitejs/plugin-react": "^4",
    "typescript": "^5",
    "@types/react": "^18",
    "@types/react-dom": "^18",
    "eslint": "^8",
    "prettier": "^3"
  }
}
```

---

## 5. 选型风险与缓解策略

| 风险 | 描述 | 缓解策略 |
|---|---|---|
| WebView2 兼容性 | 极少数 Windows 10 早期版本无内置 WebView2 | 安装包内嵌 WebView2 Runtime 引导安装 |
| git2 vendored 编译时间 | 首次编译 libgit2 较慢 | CI 缓存 `~/.cargo`，本地使用系统 libgit2 |
| specta 绑定维护 | 接口变更需重新生成类型 | 将类型生成纳入 CI，强制检查 |
| Tauri 2 生态不完整 | 部分 plugin 功能不如 Electron 丰富 | 优先使用官方 plugin，缺失功能通过 Rust command 补充 |
