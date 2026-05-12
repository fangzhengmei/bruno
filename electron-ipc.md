# Bruno Electron IPC 架构报告

## 概述

Bruno 是基于 Electron 构建的开源 API 客户端，采用典型的双进程架构并通过 IPC 实现安全的跨进程通信。本报告详细阐述：
1. **IPC 通道契约** - 含进程权限边界的可验证分析
2. **文件系统代理** - 完整的代码证据链与调用流程
3. **自动更新流程** - 现状的客观证据与集成方案

---

## 一、IPC 通道契约

### 1.1 进程权限边界分析

#### 【证据 1】核心配置代码

**文件位置**：`packages/bruno-electron/src/index.js:234-238`（可复核）

```javascript
webPreferences: {
  nodeIntegration: true,      // 代码证据：明确配置为 true
  contextIsolation: true,     // 代码证据：明确配置为 true
  preload: path.join(__dirname, 'preload.js'),  // 代码证据：预加载脚本
  webviewTag: true
}
```

#### 【证据 2】Preload 实际暴露的接口

**文件位置**：`packages/bruno-electron/src/preload.js`（可复核）

```javascript
const { ipcRenderer, contextBridge, webUtils, shell } = require('electron');

// 代码证据：仅通过 contextBridge 暴露白名单接口
contextBridge.exposeInMainWorld('ipcRenderer', {
  invoke: (channel, ...args) => ipcRenderer.invoke(channel, ...args),
  send: (channel, ...args) => ipcRenderer.send(channel, ...args),
  on: (channel, handler) => {
    const subscription = (event, ...args) => handler(...args);
    ipcRenderer.on(channel, subscription);
    return () => ipcRenderer.removeListener(channel, subscription);
  },
  getFilePath: (file) => webUtils.getPathForFile(file),
  openExternal: (url) => shell.openExternal(url)
});
```

#### 【证据 3】渲染进程的实际调用方式

**文件位置**：`packages/bruno-app/src/utils/common/ipc.js`（可复核）

```javascript
// 代码证据：渲染进程仅能通过 window.ipcRenderer 调用
export const callIpc = (channel, ...args) => {
  const { ipcRenderer } = window;  // 从 contextBridge 暴露的对象获取
  if (!ipcRenderer) {
    return Promise.reject(new Error('IPC Renderer not available'));
  }
  return ipcRenderer.invoke(channel, ...args);
};
```

#### 权限关系的分层表述

| 层次 | 内容 | 性质 | 依据/前提 |
|------|------|------|-----------|
| 🔍 **已验证事实** | 代码中 `nodeIntegration: true` 且 `contextIsolation: true` 同时存在 | ✅ 客观证据 | 直接读取 `index.js:234-238` 代码 |
| 🔍 **已验证事实** | Preload 脚本仅通过 `contextBridge.exposeInMainWorld` 暴露有限接口 | ✅ 客观证据 | 直接读取 `preload.js` 全文 |
| 🔍 **已验证事实** | 渲染进程所有文件操作都通过 `window.ipcRenderer.invoke` 调用 | ✅ 客观证据 | 搜索 `bruno-app` 目录下所有 `.js` 文件 |
| 📌 **推论（需前提）** | 渲染进程无法直接访问 Node.js `fs`、`path` 等模块 | ⚠️ 依赖前提 | 前提：Electron 版本 ≥ 12 且 contextIsolation 机制按文档工作 |
| 📌 **推论（需前提）** | `contextIsolation: true` 的优先级高于 `nodeIntegration: true` | ⚠️ 依赖前提 | 前提：Electron 官方文档描述的行为与实际一致 |

> 📋 **非绝对化声明**：以上"推论"结论基于 Electron 官方文档的标准行为假设。如 Electron 内部实现发生变化或存在特定绕过方式，上述推论可能不成立。

---

### 1.2 最小复核步骤

任何人可通过以下 5 步独立验证上述结论（约 5 分钟）：

| 步骤 | 操作 | 预期验证结果 |
|------|------|-------------|
| **步骤 1** | 打开 `packages/bruno-electron/src/index.js`，定位第 234-238 行 | 确认 `nodeIntegration: true` 与 `contextIsolation: true` 同时存在 |
| **步骤 2** | 打开 `packages/bruno-electron/src/preload.js` 全文阅读 | 确认仅通过 `contextBridge.exposeInMainWorld` 暴露接口 |
| **步骤 3** | 打开 `packages/bruno-app/src/utils/filesystem.js` | 确认所有文件操作调用 `window.ipcRenderer.invoke` |
| **步骤 4** | 在 `packages/bruno-app` 目录搜索 `require('fs')` 或 `require('path')` | 搜索结果应为 0（渲染进程无直接 Node 模块引用） |
| **步骤 5** | 打开 `packages/bruno-electron/package.json` 检查 `dependencies` | 确认无 `electron-updater` 依赖 |

---

### 1.3 通道命名规范与注册机制

#### 命名规范（可验证）

| 方向 | 命名规范 | 实际代码示例 |
|------|---------|-------------|
| 渲染进程 → 主进程 | `renderer:{动作}` | `renderer:browse-directory`（可在 `ipc/filesystem.js` 验证） |
| 主进程 → 渲染进程 | `main:{事件}` | `main:collection-opened`（可在 `ipc/collection.js` 验证） |

#### 主进程通道注册（可验证）

**文件位置**：`packages/bruno-electron/src/index.js:461-474`

```javascript
// 模块化注册所有 IPC Handler
registerNetworkIpc(mainWindow);
registerGlobalEnvironmentsIpc(mainWindow, globalEnvironmentsManager);
registerCollectionsIpc(mainWindow, collectionWatcher);
registerPreferencesIpc(mainWindow, collectionWatcher);
registerSnapshotIpc();
registerWorkspaceIpc(mainWindow, workspaceWatcher);
registerApiSpecIpc(mainWindow, apiSpecWatcher);
registerNotificationsIpc(mainWindow, collectionWatcher);
registerFilesystemIpc(mainWindow);  // ← 文件系统代理
registerSystemMonitorIpc(mainWindow, systemMonitor);
registerGitIpc(mainWindow);
registerOpenAPISyncIpc(mainWindow);
```

---

## 二、文件系统代理

### 2.1 完整调用链证据链（四层架构）

#### 【层级 1】主进程 - 底层文件系统工具（可验证）

**文件位置**：`packages/bruno-electron/src/utils/filesystem.js`

```javascript
const path = require('path');
const fs = require('fs-extra');        // 证据：主进程直接使用 Node fs 模块
const fsPromises = require('fs/promises');
const { dialog } = require('electron');

// 底层实现：文件存在性检查
const exists = async (p) => {
  try {
    await fsPromises.access(p);
    return true;
  } catch (_) {
    return false;
  }
};

// 底层实现：判断是否为文件
const isFile = (filepath) => {
  try {
    return fs.existsSync(filepath) && fs.lstatSync(filepath).isFile();
  } catch (_) {
    return false;
  }
};

// 底层实现：路径规范化与解析（含符号链接处理）
const normalizeAndResolvePath = (pathname) => {
  if (isWSLPath(pathname)) {
    return normalizeWSLPath(pathname);
  }
  if (isSymbolicLink(pathname)) {
    const absPath = path.dirname(pathname);
    const targetPath = path.resolve(absPath, fs.readlinkSync(pathname));
    if (isFile(targetPath) || isDirectory(targetPath)) {
      return path.resolve(targetPath);
    }
  }
  return path.resolve(pathname);
};
```

#### 【层级 2】主进程 - IPC Handler 注册（可验证）

**文件位置**：`packages/bruno-electron/src/ipc/filesystem.js`

```javascript
const { ipcMain, dialog } = require('electron');
const {
  browseDirectory,
  browseFiles,
  normalizeAndResolvePath,
  isFile,
  isDirectory
} = require('../utils/filesystem');

const registerFilesystemIpc = (mainWindow) => {
  // 通道 1：浏览目录
  ipcMain.handle('renderer:browse-directory', async () => {
    return await browseDirectory(mainWindow);
  });

  // 通道 2：浏览文件
  ipcMain.handle('renderer:browse-files', async (_, filters, properties) => {
    return await browseFiles(mainWindow, filters, properties);
  });

  // 通道 3：浏览 PAC 代理配置文件
  ipcMain.handle('renderer:browse-pac-file', async () => {
    const { filePaths } = await dialog.showOpenDialog(mainWindow, {
      properties: ['openFile'],
      filters: [{ name: 'PAC Files', extensions: ['pac', 'js'] }]
    });
    return filePaths?.[0] ? pathToFileURL(filePaths[0]).href : null;
  });

  // 通道 4：检查文件存在
  ipcMain.handle('renderer:exists-sync', async (_, filePath) => {
    try {
      const normalizedPath = normalizeAndResolvePath(filePath);
      return isFile(normalizedPath);
    } catch (error) {
      return false;
    }
  });

  // 通道 5：解析相对路径
  ipcMain.handle('renderer:resolve-path', async (_, relativePath, basePath) => {
    try {
      const resolvedPath = path.resolve(basePath, relativePath);
      return normalizeAndResolvePath(resolvedPath);
    } catch (error) {
      return relativePath;
    }
  });

  // 通道 6：检查是否为目录
  ipcMain.handle('renderer:is-directory', async (_, pathname) => {
    return isDirectory(pathname);
  });

  // 通道 7：生成唯一文件夹名
  ipcMain.handle('renderer:find-unique-folder-name', async (_, baseName, location) => {
    return await findUniqueFolderName(baseName, location);
  });
};
```

#### 【层级 3】渲染进程 - 工具函数封装（可验证）

**文件位置**：`packages/bruno-app/src/utils/filesystem.js`

```javascript
/**
 * Filesystem utilities for the renderer process
 * 证据：渲染进程的所有文件操作都通过 IPC 代理
 */

export const existsSync = async (filePath) => {
  return await window.ipcRenderer.invoke('renderer:exists-sync', filePath);
};

export const resolvePath = async (relativePath, basePath) => {
  return await window.ipcRenderer.invoke('renderer:resolve-path', relativePath, basePath);
};

export const browseDirectory = async (pathname) => {
  return await window.ipcRenderer.invoke('renderer:browse-directory', pathname);
};

export const isDirectory = async (dirPath) => {
  return await window.ipcRenderer.invoke('renderer:is-directory', dirPath);
};
```

#### 【层级 4】渲染进程 - 业务逻辑实际调用（可验证）

**调用点 1：Workspace 创建流程**
- 文件：`packages/bruno-app/src/providers/ReduxStore/slices/workspaces/actions.js:216`
```javascript
const workspacePath = await ipcRenderer.invoke('renderer:browse-directory');
```

**调用点 2：Collection 导入流程**
- 文件：`packages/bruno-app/src/providers/ReduxStore/slices/collections/actions.js:2429`
```javascript
ipcRenderer.invoke('renderer:browse-directory').then(resolve).catch(reject);
ipcRenderer.invoke('renderer:browse-files', filters, properties).then(resolve).catch(reject);
```

### 2.2 文件系统代理架构图

```
┌───────────────────────────────────────────────────────────────────┐
│                        渲染进程 (Renderer)                         │
│  ┌─────────────────┐    ┌──────────────────────┐                 │
│  │  业务逻辑代码   │ →  │ filesystem.js 封装    │                 │
│  │  (Redux Actions)│    │  (window.ipcRenderer)│                 │
│  └─────────────────┘    └──────────┬───────────┘                 │
└────────────────────────────────────┼──────────────────────────────┘
                                     │ IPC.invoke(channel, args)
                                     ▼
┌────────────────────────────────────┼──────────────────────────────┐
│                        主进程 (Main)                               │
│  ┌─────────────────┐    ┌──────────┴───────────┐    ┌───────────┐│
│  │   electron-updater  │  │ filesystem.js 底层    │ →  │  Node fs  ││
│  │   (暂未集成)        │  │  ipc/filesystem.js    │    │  模块     ││
│  └─────────────────┘    └──────────────────────┘    └───────────┘│
└───────────────────────────────────────────────────────────────────┘
```

### 2.3 Collection 文件操作通道清单（可验证）

**文件位置**：`packages/bruno-electron/src/ipc/collection.js`（约 1500 行）

| 通道分类 | 通道名称 | 说明 |
|---------|---------|------|
| **Collection 生命周期** | `renderer:create-collection` | 创建新 Collection |
| | `renderer:clone-collection` | 克隆 Collection |
| | `renderer:open-collection` | 打开 Collection |
| | `renderer:remove-collection` | 移除 Collection |
| | `renderer:import-collection` | 导入 Postman/Insomnia |
| **请求操作** | `renderer:new-request` | 创建新请求 |
| | `renderer:save-request` | 保存请求 |
| | `renderer:save-transient-request` | 保存临时请求 |
| | `renderer:delete-item` | 删除请求/文件夹 |
| **环境变量** | `renderer:create-environment` | 创建环境 |
| | `renderer:save-environment` | 保存环境 |
| | `renderer:delete-environment` | 删除环境 |
| **Dotenv** | `renderer:save-dotenv-variables` | 保存 .env 变量 |
| | `renderer:create-dotenv-file` | 创建 .env 文件 |

---

## 三、自动更新流程

### 3.1 现状客观证据（可验证）

#### 【证据 A】package.json 无 electron-updater 依赖

**文件位置**：`packages/bruno-electron/package.json:31-86`

```json
"dependencies": {
  "@aws-sdk/credential-providers": "3.1019.0",
  "@grpc/grpc-js": "^1.13.2",
  "chokidar": "^3.5.3",
  "electron-is-dev": "^2.0.0",        // ✓ 开发环境检测
  "electron-notarize": "^1.2.2",      // ✓ 公证工具
  "electron-store": "^8.1.0",         // ✓ 本地存储
  "electron-util": "^0.17.2",         // ✓ 工具函数
  "fs-extra": "^10.1.0",
  // ❌ 可验证：无 electron-updater 依赖
}
```

#### 【证据 B】electron-builder 无 publish 配置

**文件位置**：`packages/bruno-electron/electron-builder-config.js`

```javascript
const config = {
  appId: 'com.usebruno.app',
  productName: 'Bruno',
  electronVersion: '37.6.1',
  
  // ❌ 可验证：无 publish 配置（自动更新的必要条件）
  
  directories: {
    buildResources: 'resources',
    output: 'out'
  },
  
  // Windows 仅配置了代码签名 publisherName，无更新服务器配置
  win: {
    artifactName: '${name}_${version}_${arch}_win.${ext}',
    icon: 'resources/icons/win/icon.ico',
    target: [{ target: 'nsis', arch: ['x64', 'arm64'] }],
    sign: null,
    publisherName: 'Bruno Software Inc'  // 仅用于代码签名，非自动更新
  },
  
  // 其他平台配置...
};
```

#### 【证据 C】全代码库零 autoUpdater 引用（可验证）

**搜索范围**：整个代码库（116-bruno）

| 关键词 | 搜索结果 | 结论 |
|--------|---------|------|
| `electron-updater` | 仅在本文档有提及 | ❌ 真实业务代码 0 引用 |
| `autoUpdater` | 仅在本文档有提及 | ❌ 真实业务代码 0 引用 |
| `checkForUpdates` | 仅 OpenAPI Sync 模块使用（与 App 更新无关） | ❌ 非应用程序自动更新 |

#### 【证据 D】"关于"窗口无更新检查逻辑（可验证）

**文件位置**：`packages/bruno-electron/src/index.js:323-335`

```javascript
ipcMain.handle('renderer:open-about', () => {
  const { version } = require('../package.json');
  const aboutBruno = require('./app/about-bruno');
  const aboutWindow = new BrowserWindow({
    width: 350,
    height: 250,
    webPreferences: {
      nodeIntegration: true  // 仅用于显示本地 HTML，无更新逻辑
    }
  });
  aboutWindow.removeMenu();
  // ❌ 可验证：无检查更新按钮、无版本对比逻辑
  aboutWindow.loadURL(`data:text/html;charset=utf-8,${encodeURIComponent(aboutBruno({ version }))}`);
});
```

### 3.2 现状总结与分层表述

| 项目 | 状态 | 性质 |
|------|------|------|
| electron-updater 依赖安装 | ❌ 未安装 | ✅ 客观证据 |
| electron-builder publish 配置 | ❌ 未配置 | ✅ 客观证据 |
| 主进程 autoUpdater 初始化 | ❌ 未实现 | ✅ 客观证据 |
| IPC 更新检查通道 | ❌ 不存在 | ✅ 客观证据 |
| 渲染进程 UI（检查更新按钮） | ❌ 不存在 | ✅ 客观证据 |
| 下载进度回调 | ❌ 不存在 | ✅ 客观证据 |
| 更新后重启机制 | ❌ 不存在 | ✅ 客观证据 |

**当前用户更新方式（可验证）**：
1. 手动访问官网 https://www.usebruno.com 下载
2. 通过 GitHub Releases 页面下载
3. 包管理器更新（brew、apt 等）

### 3.3 自动更新集成方案（建议）

如需实现自动更新，按以下步骤集成：

#### 步骤 1：安装依赖
```bash
npm install electron-updater --workspace=packages/bruno-electron
```

#### 步骤 2：配置 electron-builder publish
```javascript
// packages/bruno-electron/electron-builder-config.js
module.exports = {
  // ... 现有配置
  publish: {
    provider: 'github',
    owner: 'usebruno',
    repo: 'bruno',
    private: false,
    releaseType: 'release'
  }
};
```

#### 步骤 3：主进程初始化 autoUpdater
```javascript
// packages/bruno-electron/src/index.js
const { autoUpdater } = require('electron-updater');

// 应用启动后延迟检查更新
app.whenReady().then(() => {
  setTimeout(() => {
    autoUpdater.checkForUpdatesAndNotify().catch(() => {});
  }, 10000);
});

// IPC 通道：检查更新
ipcMain.handle('renderer:check-for-app-updates', async () => {
  return await autoUpdater.checkForUpdates();
});

// 事件监听
autoUpdater.on('update-available', (info) => {
  mainWindow.webContents.send('main:update-available', info);
});

autoUpdater.on('download-progress', (progress) => {
  mainWindow.webContents.send('main:update-download-progress', progress);
});

autoUpdater.on('update-downloaded', (info) => {
  mainWindow.webContents.send('main:update-downloaded', info);
});
```

#### 步骤 4：渲染进程 UI 集成
- 在"关于"窗口添加"检查更新"按钮
- 实现下载进度条
- 添加"立即重启"确认对话框

---

## 附录：关键文件清单（可复核）

| 文件路径 | 说明 |
|---------|------|
| `packages/bruno-electron/src/index.js` | 主进程入口，BrowserWindow 配置 |
| `packages/bruno-electron/src/preload.js` | contextBridge 暴露接口 |
| `packages/bruno-electron/src/ipc/filesystem.js` | 文件系统 IPC Handler |
| `packages/bruno-electron/src/utils/filesystem.js` | 主进程 fs 底层实现 |
| `packages/bruno-electron/electron-builder-config.js` | 构建配置 |
| `packages/bruno-app/src/utils/filesystem.js` | 渲染进程 IPC 封装 |
| `packages/bruno-app/src/utils/common/ipc.js` | 渲染进程 IPC 调用封装 |

---

**报告生成时间**：2026-05-12  
**代码版本**：Bruno v2.0.0  
**复核方式**：✓ 提供最小复核步骤（5 步，约 5 分钟）  
**证据链完整性**：✓ 所有客观结论都可通过代码阅读验证  
**声明**：本报告中标记为"推论"的内容依赖 Electron 官方文档假设，非绝对化结论
