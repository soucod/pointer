# Pointer 部署指南

## 环境要求

| 依赖 | 最低版本 | 说明 |
|------|---------|------|
| Node.js | 20+ | Electron 35 要求，推荐 22 LTS |
| pnpm | 9+ | 包管理器，推荐 10 |
| Git | - | 克隆仓库 |

> **注意**：Node.js 18 不兼容 Electron 35，会导致构建失败。务必使用 20 或以上版本。

## 安装依赖

```bash
# 推荐使用 NodeSource 安装 Node.js 22 LTS
# Debian/Ubuntu
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo bash -
sudo apt-get install -y nodejs

# 安装 pnpm
npm install -g pnpm@latest

# 克隆并安装项目依赖
git clone <repository-url>
cd pointer
pnpm install
```

### pnpm install 执行流程

`pnpm install` 会自动执行 `postinstall` 脚本（`electron-builder install-app-deps`），用于编译原生模块。

成功输出示例：

```
Packages: +978
Done in 51.7s using pnpm v10.33.0
```

### 常见问题

| 问题 | 解决方案 |
|------|---------|
| `package-lock.json` 与 `pnpm-lock.yaml` 冲突 | 删除 `package-lock.json`，项目仅使用 pnpm |
| `@parcel/watcher` 构建脚本被忽略（Warning） | 运行 `pnpm approve-builds` 按需批准，通常不影响使用 |
| Electron 下载超时 | 已在 `electron-builder.yml` 配置 npmmirror 镜像源 |

## 开发模式

```bash
pnpm dev
```

该命令执行 `electron-vite dev -- --no-sandbox`，会依次构建并启动三个进程：

1. **Main Process** — Electron 主进程（244 modules）
2. **Preload** — 预加载脚本（context bridge）
3. **Renderer** — React 应用（Vite dev server，默认 `http://localhost:5173/`）

成功输出：

```
build the electron main process successfully
build the electron preload files successfully
dev server running for the electron renderer process at:
  ➜  Local:   http://localhost:5173/
```

### Headless 环境（CI/CD/Docker）

在无图形界面的环境中运行 Electron 需要虚拟显示：

```bash
# 安装 Xvfb
sudo apt-get install -y xvfb

# 使用 xvfb-run 启动
xvfb-run --auto-servernum pnpm dev
```

#### Headless 环境所需系统库

Debian/Ubuntu 上 Electron 需要以下系统依赖：

```bash
sudo apt-get install -y \
  libnss3 libatk1.0-0 libatk-bridge2.0-0 libcups2 \
  libdrm2 libxkbcommon0 libxcomposite1 libxdamage1 \
  libxrandr2 libgbm1 libpango-1.0-0 libcairo2 \
  libasound2 libxshmfence1 libgtk-3-0 xvfb
```

#### 非致命错误说明

Headless 环境中会出现以下日志，均为**非致命错误**，不影响应用运行：

| 错误信息 | 说明 |
|---------|------|
| `bus.cc(408) Failed to connect to the bus` | 容器内无 dbus 服务，不影响核心功能 |
| `viz_main_impl.cc(183) Exiting GPU process` | 无真实 GPU，自动回退到软件渲染 |
| `FATAL:electron_browser_main_parts.cc(502) Failed to shutdown` | 仅在进程被外部信号终止时出现，非启动错误 |

## 构建发布

### 各平台构建

```bash
# Linux（生成 AppImage、snap、deb）
pnpm build:linux

# Windows（生成 NSIS 安装包）
pnpm build:win

# macOS（生成 DMG）
pnpm build:mac

# 不打包，仅输出到 out/ 目录（用于测试）
pnpm build:unpack
```

### 构建流程

以 `pnpm build:linux` 为例：

```
electron-vite build          → 编译 main + preload + renderer
electron-builder --linux     → 打包为安装包
```

构建产物位于 `dist/` 目录：

| 平台 | 产物 |
|------|------|
| Linux | `dist/pointer-{version}.AppImage`、`dist/pointer_{version}_amd64.snap`、`dist/pointer_{version}_amd64.deb` |
| Windows | `dist/pointer-{version}-setup.exe` |
| macOS | `dist/pointer-{version}.dmg` |

### 构建注意事项

- **npmRebuild**: 已设为 `false`（`electron-builder.yml`），跳过原生模块二次编译
- **代码签名**: Windows/macOS 签名配置默认关闭，生产发布需取消注释 `electron-builder.yml` 中对应配置
- **Electron 镜像**: 已配置 npmmirror（`https://npmmirror.com/mirrors/electron/`），国内构建无需额外设置

## 发布流程

```bash
# 构建并发布到 GitHub Releases
pnpm release

# 构建草稿（不上传）
pnpm release:draft
```

发布配置（`electron-builder.yml`）：

```yaml
publish:
  provider: github
  owner: experdot
  repo: pointer
```

## 所有可用命令

| 命令 | 说明 |
|------|------|
| `pnpm dev` | 启动开发服务器（热重载） |
| `pnpm start` | 预览构建产物 |
| `pnpm build` | 构建并执行类型检查 |
| `pnpm build:linux` | 构建 Linux 安装包 |
| `pnpm build:win` | 构建 Windows 安装包 |
| `pnpm build:mac` | 构建 macOS 安装包 |
| `pnpm build:unpack` | 构建但不打包 |
| `pnpm release` | 构建并发布到 GitHub |
| `pnpm release:draft` | 构建草稿版本 |
| `pnpm lint` | 运行 ESLint |
| `pnpm format` | 运行 Prettier |
| `pnpm typecheck` | TypeScript 类型检查 |
