# Bruno Electron IPC 架构报告

## 概述

Bruno 是一个基于 Electron 构建的开源 API 客户端，采用典型的 Electron 双进程架构：
- **主进程（Main Process）**：负责文件系统操作、本地存储、网络请求、系统集成
- **渲染进程（Renderer Process）**：负责用户界面交互、状态管理
- **IPC（Inter-Process Communication）**：进程间通信桥梁

---

## 一、IPC 通道契约

### 1.1 架构设计原则

**安全隔离**
- 渲染进程运行在沙箱环境中，不直接访问 Node.js API
- 通过 `contextBridge` 暴露有限的接口给渲染进程
- `nodeIntegration: true`, `contextIsolation: true`

**通道命名规范**
- 渲染进程 → 主进程：`renderer:{action}`
- 主进程 → 渲染进程：`main:{event}`

### 1.2 Preload 脚本暴露接口

**文件位置**：`packages/bruno-electron/src/preload.js`

```javascript
contextBridge.exposeInMainWorld('ipcRenderer', {
  // 调用主进程 Handler（Promise）
  invoke: (channel, ...args) => ipcRenderer.invoke(channel, ...args),
  
  // 发送单向消息
  send: (channel, ...args) => ipcRenderer.send(channel, ...args),
  
  // 监听主进程事件（支持清理）
  on: (channel, handler) => {
    const subscription = (event, ...args) => handler(...args);
    ipcRenderer.on(channel, subscription);
    return () => ipcRenderer.removeListener(channel, subscription);
  },
  
  // 获取文件路径（File API 桥接）
  getFilePath: (file) => webUtils.getPathForFile(file),
  
  // 打开外部链接
  openExternal: (url) => shell.openExternal(url)
});
```

### 1.3 主要 IPC 通道分类

#### 窗口控制通道
| 通道 | 类型 | 说明 |
|------|------|------|
| `renderer:window-minimize` | send | 最小化窗口 |
| `renderer:window-maximize` | send | 最大化/还原窗口 |
| `renderer:window-close` | send | 关闭窗口 |
| `renderer:window-is-maximized` | invoke | 查询窗口状态 |
| `renderer:toggle-fullscreen` | invoke | 切换全屏 |

#### 偏好设置通道
| 通道 | 类型 | 说明 |
|------|------|------|
| `renderer:ready` | invoke | 渲染进程就绪通知 |
| `renderer:save-preferences` | invoke | 保存用户偏好 |
| `renderer:theme-change` | send | 主题变更通知 |
| `renderer:get-system-proxy-variables` | invoke | 获取系统代理配置 |
| `main:load-preferences` | on | 偏好设置加载完成 |
| `main:open-preferences` | on | 打开偏好设置面板 |

#### 缩放控制通道
| 通道 | 类型 | 说明 |
|------|------|------|
| `renderer:reset-zoom` | invoke | 重置缩放 |
| `renderer:zoom-in` | invoke | 放大 |
| `renderer:zoom-out` | invoke | 缩小 |
| `renderer:set-zoom-level` | invoke | 设置缩放级别 |

---

## 二、文件系统代理

### 2.1 设计理念

渲染进程不直接操作文件系统，所有文件操作通过 IPC 代理到主进程执行。

### 2.2 核心文件系统 IPC Handlers

**文件位置**：`packages/bruno-electron/src/ipc/filesystem.js`

#### 目录浏览
```javascript
// 打开目录选择对话框
ipcMain.handle('renderer:browse-directory', async () => {
  return await browseDirectory(mainWindow);
});

// 打开文件选择对话框
ipcMain.handle('renderer:browse-files', async (_, filters, properties) => {
  return await browseFiles(mainWindow, filters, properties);
});
```

#### 路径操作
```javascript
// 检查文件是否存在
ipcMain.handle('renderer:exists-sync', async (_, filePath) => {
  const normalizedPath = normalizeAndResolvePath(filePath);
  return isFile(normalizedPath);
});

// 解析相对路径
ipcMain.handle('renderer:resolve-path', async (_, relativePath, basePath) => {
  const resolvedPath = path.resolve(basePath, relativePath);
  return normalizeAndResolvePath(resolvedPath);
});

// 检查是否为目录
ipcMain.handle('renderer:is-directory', async (_, pathname) => {
  return isDirectory(pathname);
});
```

### 2.3 Collection 文件操作（核心业务）

**文件位置**：`packages/bruno-electron/src/ipc/collection.js`

#### Collection 生命周期操作

| 通道 | 说明 |
|------|------|
| `renderer:create-collection` | 创建新 Collection |
| `renderer:clone-collection` | 克隆 Collection |
| `renderer:rename-collection` | 重命名 Collection |
| `renderer:open-collection` | 打开 Collection 对话框 |
| `renderer:open-multiple-collections` | 批量打开 Collection |
| `renderer:remove-collection` | 移除 Collection |
| `renderer:import-collection` | 导入 Collection |

#### 请求/文件夹 CRUD 操作

| 通道 | 说明 |
|------|------|
| `renderer:new-request` | 创建新请求 |
| `renderer:save-request` | 保存请求 |
| `renderer:new-folder` | 创建新文件夹 |
| `renderer:delete-item` | 删除文件/文件夹 |
| `renderer:rename-item-filename` | 重命名项 |
| `renderer:rename-item-name` | 重命名显示名称 |

#### 环境变量操作

| 通道 | 说明 |
|------|------|
| `renderer:create-environment` | 创建环境 |
| `renderer:save-environment` | 保存环境 |
| `renderer:rename-environment` | 重命名环境 |
| `renderer:delete-environment` | 删除环境 |
| `renderer:update-environment-color` | 更新环境颜色 |
| `renderer:save-dotenv-variables` | 保存 .env 变量 |
| `renderer:create-dotenv-file` | 创建 .env 文件 |

#### 主进程事件通知

| 事件 | 说明 |
|------|------|
| `main:collection-opened` | Collection 已打开 |
| `main:collection-renamed` | Collection 已重命名 |
| `main:collection-import-started` | Collection 导入开始 |
| `main:collection-import-ended` | Collection 导入结束 |

### 2.4 文件格式支持

- **Bru 格式**：`.bru` 扩展名，Bruno 原生格式
- **YAML 格式**：`.yml` 扩展名，OpenCollection 格式
- **Environment 文件**：`environments/` 目录下
- **Dotenv 文件**：`.env` 文件支持

---

## 三、自动更新流程

### 3.1 当前状态

**注意**：经过代码库全面检查，Bruno 当前版本 **未集成** `electron-updater` 自动更新机制。

### 3.2 发布构建配置

**文件位置**：`packages/bruno-electron/electron-builder-config.js`

#### 支持平台与分发格式

| 平台 | 分发格式 | 说明 |
|------|----------|------|
| **macOS** | `.dmg`, `.pkg`, `.zip` | 支持 x64 和 arm64 架构 |
| **Windows** | `.exe` (NSIS) | 支持 x64 和 arm64 架构 |
| **Linux** | `.AppImage`, `.deb`, `.rpm` | 支持 x64 和 arm64 架构 |

#### 构建配置要点

```javascript
{
  appId: 'com.usebruno.app',
  productName: 'Bruno',
  electronVersion: '37.6.1',
  
  // macOS 配置
  mac: {
    hardenedRuntime: true,           // 硬化运行时
    notarize: false,                  // 公证（可配置）
    protocols: [{ name: 'Bruno', schemes: ['bruno'] }]  // URL Scheme
  },
  
  // Windows 配置
  win: {
    target: [{ target: 'nsis', arch: ['x64', 'arm64'] }],
    publisherName: 'Bruno Software Inc'
  },
  
  // NSIS 安装程序配置
  nsis: {
    oneClick: false,                  // 非一键安装
    allowToChangeInstallationDirectory: true,  // 允许选择目录
    allowElevation: true,             // 允许提升权限
    createDesktopShortcut: true,
    createStartMenuShortcut: true
  }
}
```

### 3.3 更新方式（当前）

用户通过以下方式手动更新：
1. 访问官方网站 https://www.usebruno.com 下载最新版本
2. 通过 GitHub Releases 页面下载
3. 包管理器更新（如 brew, apt 等）

### 3.4 潜在的自动更新集成方案

如需实现自动更新，可按以下方案集成 `electron-updater`：

#### 步骤 1：安装依赖
```bash
npm install electron-updater --save
```

#### 步骤 2：配置 publish 选项
```javascript
// electron-builder-config.js
module.exports = {
  // ... 现有配置
  publish: {
    provider: 'github',
    owner: 'usebruno',
    repo: 'bruno',
    private: false
  }
};
```

#### 步骤 3：主进程集成
```javascript
// packages/bruno-electron/src/index.js
const { autoUpdater } = require('electron-updater');

// 检查更新
function checkForUpdates() {
  autoUpdater.checkForUpdatesAndNotify();
}

// IPC 通道：检查更新
ipcMain.handle('renderer:check-for-updates', async () => {
  return autoUpdater.checkForUpdates();
});

// 事件监听
autoUpdater.on('update-available', (info) => {
  mainWindow.webContents.send('main:update-available', info);
});

autoUpdater.on('update-downloaded', (info) => {
  mainWindow.webContents.send('main:update-downloaded', info);
});
```

#### 步骤 4：渲染进程 UI 集成
- 添加"检查更新"菜单项
- 实现更新通知弹窗
- 提供下载进度显示

---

## 四、其他重要 IPC 模块

### 4.1 网络请求模块
- `renderer:run-request` - 执行 HTTP 请求
- `renderer:cancel-request` - 取消请求
- `main:run-request-event` - 请求进度事件
- 支持 gRPC、WebSocket 等协议

### 4.2 Workspace 模块
- `renderer:create-workspace` - 创建工作空间
- `renderer:open-workspace` - 打开工作空间
- `renderer:save-workspace-docs` - 保存工作空间文档
- `main:workspace-opened` - 工作空间打开通知

### 4.3 Git 集成模块
- `renderer:git-init` - 初始化 Git 仓库
- `renderer:git-commit` - 提交更改
- `renderer:git-push-pull` - 推送/拉取
- `main:update-git-operation-progress` - Git 进度通知

### 4.4 系统监控模块
- `renderer:start-system-monitoring` - 开始系统监控
- `renderer:stop-system-monitoring` - 停止系统监控
- 监控 CPU、内存等系统资源

### 4.5 OpenAPI Sync 模块
- `renderer:check-openapi-updates` - 检查 OpenAPI 更新
- `renderer:apply-openapi-sync` - 应用同步变更
- `renderer:get-collection-drift` - 获取变更差异

---

## 五、架构总结

### 5.1 优势
1. **职责清晰**：主进程负责系统资源，渲染进程专注 UI
2. **安全性**：通过 contextBridge 限制暴露接口
3. **可扩展**：模块化的 IPC Handler 设计
4. **类型安全**：通道命名规范减少出错概率

### 5.2 技术栈
- **Electron 37.6.1** - 应用框架
- **electron-builder** - 构建与打包
- **electron-store** - 本地存储
- **chokidar** - 文件系统监听

### 5.3 目录结构
```
packages/bruno-electron/src/
├── index.js              # 主进程入口
├── preload.js            # 预加载脚本
├── ipc/                  # IPC Handler 模块
│   ├── filesystem.js     # 文件系统代理
│   ├── collection.js     # Collection 操作
│   ├── network/          # 网络请求处理
│   ├── workspace.js      # 工作空间
│   ├── preferences.js    # 偏好设置
│   ├── git.js            # Git 集成
│   └── ...
├── store/                # 状态存储
├── utils/                # 工具函数
└── app/                  # 应用逻辑
```

---

**报告生成时间**：2026-05-12
**代码版本**：Bruno v2.0.0
