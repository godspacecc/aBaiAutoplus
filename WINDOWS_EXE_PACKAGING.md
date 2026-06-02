# Windows EXE 打包说明

本文档记录如何把本项目打包成 Windows 桌面安装程序。项目的桌面版不是单纯的 Python exe，而是：

- `main.py`：FastAPI 后端入口，生产模式下托管前端静态文件。
- `frontend/`：React + Vite 前端，构建产物输出到根目录 `static/`。
- `electron/`：Electron 桌面壳，生产模式下启动内置后端 exe，再打开 `http://localhost:8000`。
- `electron/electron-builder.yml`：Electron Builder 配置，Windows 产物输出到 `electron/dist/`。

## 打包产物

Windows 打包完成后，目标产物为：

```text
electron/dist/*.exe
```

该 exe 是 Electron Builder 生成的 NSIS 安装包。安装后，应用会从 Electron resources 目录启动内置 Python 后端。

## 环境要求

建议在 Windows 上打包 Windows 版本。

- Windows 10/11
- Python 3.12
- Node.js 20
- PowerShell
- 可访问 npm / Electron 下载源

如果国内网络下载 Electron 较慢，可以使用镜像环境变量：

```powershell
$env:ELECTRON_MIRROR = "https://npmmirror.com/mirrors/electron/"
$env:ELECTRON_BUILDER_BINARIES_MIRROR = "https://npmmirror.com/mirrors/electron-builder-binaries/"
```

## 一次完整打包流程

在项目根目录执行：

```powershell
cd F:\GitHub\aBaiAutoplus
```

创建并安装 Python 依赖：

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\python -m pip install -U pip
.\.venv\Scripts\pip install -r requirements.txt
.\.venv\Scripts\pip install pyinstaller
```

构建前端。`frontend/vite.config.ts` 已配置 `outDir: '../static'`，所以构建后会生成根目录 `static/`：

```powershell
cd frontend
npm ci --legacy-peer-deps
npm run build
cd ..
```

打包 Python 后端。Windows 下 PyInstaller 的 `--add-data` 分隔符必须使用分号 `;`：

```powershell
.\.venv\Scripts\pyinstaller --onedir --name backend `
  --add-data "platforms;platforms" `
  --add-data "core;core" `
  --add-data "api;api" `
  --add-data "services;services" `
  --add-data "providers;providers" `
  --add-data "application;application" `
  --add-data "infrastructure;infrastructure" `
  --add-data "domain;domain" `
  --add-data "static;static" `
  --collect-all browserforge `
  --collect-all camoufox `
  --collect-all apify_fingerprint_datapoints `
  --collect-all language_tags `
  --collect-all ua_parser `
  --collect-all ua_parser_builtins `
  --collect-all quart `
  --collect-all hypercorn `
  --collect-all aiofiles `
  --collect-all werkzeug `
  main.py
```

复制后端产物到 Electron 打包目录：

```powershell
New-Item -ItemType Directory -Force -Path electron\backend
Copy-Item -Recurse -Force dist\backend electron\backend\
```

构建 Windows 安装包：

```powershell
cd electron
npm ci --legacy-peer-deps
npx electron-builder --win --publish never
```

完成后查看：

```powershell
Get-ChildItem .\dist\*.exe
```

## 自动发布流程

仓库已有 GitHub Actions 发布流程：

```text
.github/workflows/release.yml
```

推送 `v*` tag 会触发自动构建并发布 Release：

```powershell
git tag v1.0.0
git push origin v1.0.0
```

该流程会分别构建 macOS、Windows 和 Docker 镜像。Windows 产物会上传为 `electron/dist/*.exe`。

## 运行机制

Electron 生产模式下执行以下逻辑：

1. 从 `process.resourcesPath/backend/backend/backend.exe` 找到 PyInstaller 后端。
2. 启动后端进程，监听 `8000` 端口。
3. 轮询 `http://localhost:8000/api/health`。
4. 后端就绪后打开主窗口，加载 `http://localhost:8000`。
5. 应用退出时，Windows 下通过 `taskkill /pid <pid> /T /F` 结束后端进程树。

对应代码在 `electron/main.js`。

## 常见问题

### 1. 找不到 icon.ico

`electron/electron-builder.yml` 配置了：

```yaml
win:
  icon: build/icon.ico
```

如果打包时报 icon 缺失，需要添加：

```text
electron/build/icon.ico
```

或者临时移除 `win.icon` 配置。

### 2. 后端启动失败或 Electron 一直停在启动页

先单独运行 PyInstaller 产物验证：

```powershell
.\dist\backend\backend.exe
```

然后访问：

```text
http://localhost:8000/api/health
```

如果健康检查不通，优先检查：

- `8000` 端口是否被占用。
- 防火墙是否拦截本地监听。
- PyInstaller 是否漏收动态模块或数据目录。
- `static/` 是否已经由前端构建生成。

### 3. 前端页面空白

确认根目录存在 `static/index.html`：

```powershell
Test-Path .\static\index.html
```

如果不存在，重新执行：

```powershell
cd frontend
npm run build
cd ..
```

### 4. Windows 下 add-data 写法错误

Windows 使用：

```powershell
--add-data "源路径;目标路径"
```

macOS / Linux 使用：

```bash
--add-data "源路径:目标路径"
```

不要在 Windows PowerShell 中直接使用 `electron/build-backend.sh`，该脚本使用的是 Unix 路径和分隔符。

### 5. 浏览器自动化能力缺依赖

如果打包后的协议或浏览器功能运行异常，需要检查相关依赖是否被 PyInstaller 收集。当前命令已显式收集：

- `browserforge`
- `camoufox`
- `apify_fingerprint_datapoints`
- `language_tags`
- `ua_parser`
- `ua_parser_builtins`
- `quart`
- `hypercorn`
- `aiofiles`
- `werkzeug`

如果新增平台或 provider 使用了动态导入，可能需要继续补充 `--hidden-import` 或 `--collect-all`。

## 清理旧产物

重新打包前可以清理：

```powershell
Remove-Item -Recurse -Force dist, build, backend.spec, electron\backend, electron\dist -ErrorAction SilentlyContinue
```

注意不要清理 `.venv/`、`frontend/node_modules/`、`electron/node_modules/`，除非需要完全重装依赖。
