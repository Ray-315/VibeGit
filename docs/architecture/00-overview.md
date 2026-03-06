# VibeGit — 可视化 Git 版本管理桌面工具 · 完整架构设计规格书

> **版本**: v2.0  
> **日期**: 2026-03-06  
> **定位**: 产品级架构设计文档 + Codex 可执行实施蓝图  
> **作者角色**: 资深桌面应用架构师 / Rust-Tauri 技术负责人 / 产品级前端系统设计师  
> **核心能力**: 本地 Git 版本管理 + GitHub 平台集成（OAuth 登录、Push/Pull/Fetch、PR 管理）

---

## 文档结构

本架构设计规格书分为以下章节，每个章节为独立文件：

| 文件 | 章节 | 内容 |
|---|---|---|
| [01-tech-selection.md](./01-tech-selection.md) | A. 技术路线评估与最终选型 | 方案对比、选型理由、依赖清单 |
| [02-product-scope.md](./02-product-scope.md) | B. 产品范围定义 | MVP 功能、非目标、扩展规划 |
| [03-system-architecture.md](./03-system-architecture.md) | C. 系统总体架构 | 分层架构、通信方式、事件总线、异步策略 |
| [04-directory-structure.md](./04-directory-structure.md) | D. 目录结构设计 | 完整目��树、职责说明 |
| [05-domain-models.md](./05-domain-models.md) | E. Git 领域模型设计 | 所有 Rust 数据结构 + TypeScript 类型 |
| [06-rust-backend.md](./06-rust-backend.md) | F. Rust 后端设计 | 模块划分、错误设计、线程模型、OAuth、凭证、Remote |
| [07-frontend-architecture.md](./07-frontend-architecture.md) | G. 前端架构设计 | 状态管理、命令封装、布局、动效、主题 |
| [08-hidpi-ui.md](./08-hidpi-ui.md) | H. 高分屏与 UI 适配设计 | DPI scaling、布局原则、桌面原生感 |
| [09-module-breakdown.md](./09-module-breakdown.md) | I. 功能模块拆分 | 模块依赖关系、MVP 标注 |
| [10-api-design.md](./10-api-design.md) | J. Tauri 命令 API 设计 | 所有命令的完整规格 |
| [11-edge-cases.md](./11-edge-cases.md) | K. 边界情况与风险清单 | 异常场景、处理策略 |
| [12-roadmap.md](./12-roadmap.md) | L. 开发路线图 | 四阶段开发计划 |
| [13-codex-instructions.md](./13-codex-instructions.md) | M. Codex 可执行输出 | 给 AI 的工程实施指令 |
| [14-parallel-agents.md](./14-parallel-agents.md) | N. 多 Agent 并行开发指南 | 分支策略、Agent 数量与分工、每个 Agent 的详细指令 |

---

## 项目一句话定位

> 一个高颜值、现代化、支持高分辨率屏幕缩放的 Git 可视化桌面工具，基于 Tauri 2 + Rust + React 构建，支持本地 Git 管理和 GitHub 平台集成，面向想用图形界面管理 Git 的开发者。

## 技术栈速览

```
后端：Tauri 2 + Rust + git2 + reqwest + keyring
前端：React + TypeScript + Tailwind CSS + shadcn/ui + Motion
状态：Zustand (UI) + TanStack Query (数据)
构建：Vite + pnpm
```

## 视觉气质

- 克制但高级的层次感
- 舒服的留白
- 清晰的面板分区
- 适度的圆角、阴影
- 高对比但不刺眼的暗色主题（默认）
- 动效短、稳、轻
- 避免"表格后台管理系统感"