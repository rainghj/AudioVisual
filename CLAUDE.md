# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

基于 Electron 33 的视频解析工具，支持腾讯视频、爱奇艺、优酷、哔哩哔哩、芒果TV 等国内平台及美韩日剧站点。核心机制是通过注入第三方解析接口的 iframe 播放器来绕过平台限制播放视频。

## Architecture

### 进程模型

- **主进程** (`main.js`): Electron 主进程，管理窗口、BrowserView、IPC 通信、自动更新、Widevine CDM
- **渲染进程** (`renderer.js`): UI 逻辑，运行在 `index.html` 中，通过 `preload-ui.js` 桥接 IPC
- **Web 预加载** (`preload-web.js`): 注入到 BrowserView 的网页中，负责播放器注入、样式隐藏、反 Bot 检测

### 核心机制

- **BrowserView 池** (`viewPool`): 预渲染美韩日剧站点并缓存，切换时直接激活而非重新加载
- **iframe 注入引擎** (`startInjectionGuardian`): 在目标网页中以 50ms 高频轮询寻找视频容器，注入解析 iframe
- **CSS 注入系统**: 主进程注入主题 CSS + 滚动条样式 + 隐藏弹窗/广告的 nuisance CSS
- **主题系统**: 国内模式（深蓝 `#1e1e2f`）和美韩日剧模式（纯黑 `#000000`），通过 CSS 变量切换
- **窗口状态持久化**: 位置、大小、侧边栏折叠状态保存到 `userData/window-state.json`

### 解析流程

1. 用户在侧边栏选择平台 → 2. 导航到平台首页 → 3. 用户点击具体视频 → 4. 选择解析接口 → 5. 点击 Parse! → 6. 将视频 URL 与解析 API 拼接 → 7. 通过 IPC 发送到主进程 → 8. 主进程向 BrowserView 发送 `apply-embed-video` → 9. preload-web.js 的 Guardian 开始轮询注入解析 iframe

### 文件结构

```
├── main.js                    # Electron 主进程
├── index.html                 # 主界面 UI
├── package.json               # 项目配置与构建脚本
├── assets/
│   ├── css/
│   │   ├── style.css          # 国内模式样式（深蓝主题）
│   │   ├── drama-style.css    # 美韩日剧模式样式（纯黑主题）
│   │   ├── view-style.css     # BrowserView 滚动条样式
│   │   └── toastify.min.css   # Toast 通知样式
│   ├── js/
│   │   ├── renderer.js        # 渲染进程（UI 逻辑）
│   │   ├── preload-ui.js      # 主窗口预加载 (contextBridge -> voidAPI)
│   │   ├── preload-web.js     # BrowserView 预加载（注入引擎）
│   │   └── toastify.min.js    # Toast 通知库
│   ├── fonts/
│   │   └── HarmonyOS_SansSC_Medium.ttf
│   └── images/
├── .github/workflows/build.yml  # CI: tag 推送后构建 macOS/Linux 并发布 GitHub Release
└── userData/                  # 运行时数据（gitignored）
```

## Key Technical Details

- **Widevine CDM**: 自动从 Chrome 安装目录检测并加载 Widevine CDM，支持 DRM 内容
- **安全策略**: 所有窗口 `contextIsolation: true`, `nodeIntegration: false`；移除 CSP 和 X-Frame-Options 响应头以允许 iframe 嵌套
- **用户设置**: 解析接口列表和美韩日剧站点列表通过 `localStorage` 持久化
- **自动更新**: 基于 `electron-updater` 连接 GitHub Releases，手动触发检查，30 秒超时保护

## Development Commands

```bash
# 安装依赖
npm install

# 开发启动
npm start

# 构建分发
npm run dist              # 当前平台
npm run dist:win          # Windows (NSIS)
npm run dist:mac          # macOS (DMG, x64 + arm64)
npm run dist:linux        # Linux (deb)
npm run dist:all          # 全平台
```

## Important Constraints

- 美韩日剧模式最多只能添加 4 个站点（renderer.js 中硬限制）
- 窗口最小尺寸 940x620
- 国内模式侧边栏宽度 `clamp(200px, 18vw, 280px)`，顶部工具栏 `clamp(50px, 7vh, 65px)`
- BrowserView 的缩放因子基于 `viewWidth / 1400` 计算
- 预渲染缓存有效期 24 小时（`cache_info.json`）
