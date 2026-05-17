# Bruno 文件树监听防抖实现说明

## 1. 概述

Bruno 使用多层级的文件监听与防抖机制，确保文件系统变更能够准确、高效地反映到 UI 上，同时避免重复渲染和性能问题。整个实现分为六个核心层面：**变更感知层**、**事件合并层**、**刷新控制层**、**去重加载层**、**集合路径去重层**、**快照恢复层**。

## 2. 变更感知层 (Chokidar Watcher)

### 2.1 核心配置

文件监听基于 `chokidar` 库，在 `packages/bruno-electron/src/app/collection-watcher.js` 中配置：

```javascript
const watcher = chokidar.watch(watchPath, {
  ignoreInitial: false,
  usePolling: isWSLPath(watchPath) || forcePolling ? true : false,
  ignored: (filepath) => { /* 忽略逻辑 */ },
  persistent: true,
  ignorePermissionErrors: true,
  awaitWriteFinish: {
    stabilityThreshold: 80,  // 防抖阈值：80ms 无变更视为写入完成
    pollInterval: 10        // 轮询间隔：10ms
  },
  depth: 20,
  disableGlobbing: true
});
```

### 2.2 关键防抖机制

`awaitWriteFinish` 是第一层防抖：
- **stabilityThreshold: 80ms**：文件在 80ms 内没有进一步写入时，才触发 change 事件
- **pollInterval: 10ms**：每 10ms 检查一次文件状态
- 这避免了文件写入过程中（如编辑器自动保存）触发多次变更事件

### 2.3 忽略规则

```javascript
// 始终忽略 node_modules 和 .git
const defaultIgnores = ['node_modules', '.git'];

// 忽略 .env 文件（由专门的 dotenv-watcher 处理）
if (basename === '.env' || basename.startsWith('.env.')) {
  return true;
}

// 检查路径分段是否匹配忽略模式
const pathSegments = relativePath.split(path.sep);
if (pathSegments.some((segment) => defaultIgnores.includes(segment))) {
  return true;
}
```

### 2.4 监听事件类型

```javascript
watcher
  .on('ready', () => onWatcherSetupComplete(...))   // 初始扫描完成
  .on('add', (pathname) => add(...))                // 文件新增
  .on('addDir', (pathname) => addDirectory(...))    // 目录新增
  .on('change', (pathname) => change(...))          // 文件变更
  .on('unlink', (pathname) => unlink(...))          // 文件删除
  .on('unlinkDir', (pathname) => unlinkDir(...))    // 目录删除
  .on('error', (error) => { /* 错误处理 */ });
```

## 3. 事件合并层 (Redux Actions)

### 3.1 事件分发流程

主进程监听到文件变更后，通过 IPC 发送到渲染进程：

```javascript
// packages/bruno-electron/src/app/collection-watcher.js
win.webContents.send('main:collection-tree-updated', 'addFile', file);
win.webContents.send('main:collection-tree-updated', 'change', file);
win.webContents.send('main:collection-tree-updated', 'unlink', file);
```

渲染进程在 `packages/bruno-app/src/providers/App/useIpcEvents.js` 中接收并分发 Redux action：

```javascript
const _collectionTreeUpdated = (type, val) => {
  if (type === 'addFile') {
    dispatch(collectionAddFileEvent({ file: val }));
  }
  if (type === 'change') {
    dispatch(collectionChangeFileEvent({ file: val }));
  }
  if (type === 'unlink') {
    setTimeout(() => {  // 删除操作额外延迟 100ms，避免与重命名冲突
      dispatch(collectionUnlinkFileEvent({ file: val }));
    }, 100);
  }
};
```

### 3.2 UID 保留机制（核心合并策略）

为避免 React 组件因 key 变化而重新渲染，实现了 UID 保留机制：

```javascript
// packages/bruno-app/src/providers/ReduxStore/slices/collections/index.js

/**
 * 合并时保留现有数组项的 UID，避免 React key 不稳定导致的重新渲染
 */
const preserveUidsAtPaths = (existing, updated, paths) => {
  if (!existing || !updated) return updated;
  const merged = cloneDeep(updated);

  paths.forEach((path) => {
    const newArray = get(merged, path);
    const existingArray = get(existing, path, []);

    if (Array.isArray(newArray) && newArray.length) {
      set(merged, path, newArray.map((item, i) => 
        existingArray[i]?.uid ? { ...item, uid: existingArray[i].uid } : item
      ));
    }
  });
  return merged;
};

// 请求级别 UID 保留路径
const REQUEST_UID_PATHS = [
  'params', 'headers', 'vars.req', 'vars.res', 
  'assertions', 'body.formUrlEncoded', 'body.multipartForm', 'body.file'
];

// 集合根级别 UID 保留路径
const ROOT_UID_PATHS = ['request.headers', 'request.vars.req', 'request.vars.res'];

const mergeRequestWithPreservedUids = (existingRequest, newRequest) =>
  preserveUidsAtPaths(existingRequest, newRequest, REQUEST_UID_PATHS);

const mergeRootWithPreservedUids = (existingRoot, newRoot) =>
  preserveUidsAtPaths(existingRoot, newRoot, ROOT_UID_PATHS);
```

### 3.3 变更事件处理

```javascript
collectionChangeFileEvent: (state, action) => {
  const { file } = action.payload;
  const collection = findCollectionByUid(state.collections, file.meta.collectionUid);
  
  if (collection) {
    const item = findItemInCollection(collection, file.data.uid);
    
    if (item) {
      // 检查是否只是排序变更（seq 更新）
      if (areItemsTheSameExceptSeqUpdate(item, file.data)) {
        item.seq = file.data.seq;
        // 草稿也同步更新 seq
        if (item?.draft) {
          item.draft.seq = file.data.seq;
        }
        // 如果草稿与文件内容一致，清除草稿
        if (item?.draft && areItemsTheSameExceptSeqUpdate(item?.draft, file.data)) {
          item.draft = null;
        }
      } else {
        // 完整更新，但保留 UID
        item.name = file.data.name;
        item.type = file.data.type;
        item.seq = file.data.seq;
        item.tags = file.data.tags;
        item.request = mergeRequestWithPreservedUids(item.request, file.data.request);
        item.settings = file.data.settings;
        item.examples = file.data.examples;
        
        // 仅当草稿与文件内容完全一致时才清除，保护用户正在编辑的内容
        if (item.draft && areItemsTheSameExceptSeqUpdate(item.draft, file.data)) {
          item.draft = null;
        }
      }
    }
  }
}
```

## 4. 刷新控制层 (Loading State Machine)

### 4.1 加载状态跟踪

为避免初始扫描时的重复渲染，实现了加载状态机：

```javascript
// packages/bruno-electron/src/app/collection-watcher.js
class CollectionWatcher {
  constructor() {
    this.watchers = {};
    this.loadingStates = {};  // 每个集合的加载状态
    this.tempDirectoryMap = {};
  }

  // 初始化加载状态
  initializeLoadingState(collectionUid) {
    if (!this.loadingStates[collectionUid]) {
      this.loadingStates[collectionUid] = {
        isDiscovering: false,   // 初始发现阶段
        isProcessing: false,    // 文件处理阶段
        pendingFiles: new Set() // 待处理文件集合
      };
    }
  }

  // 开始发现阶段
  startCollectionDiscovery(win, collectionUid) {
    this.initializeLoadingState(collectionUid);
    const state = this.loadingStates[collectionUid];
    state.isDiscovering = true;
    state.pendingFiles.clear();
    
    // 通知 UI 显示加载状态
    win.webContents.send('main:collection-loading-state-updated', {
      collectionUid,
      isLoading: true
    });
  }

  // 添加文件到待处理队列
  addFileToProcessing(collectionUid, filepath) {
    this.initializeLoadingState(collectionUid);
    const state = this.loadingStates[collectionUid];
    state.pendingFiles.add(filepath);
  }

  // 标记文件处理完成
  markFileAsProcessed(win, collectionUid, filepath) {
    if (!this.loadingStates[collectionUid]) return;
    
    const state = this.loadingStates[collectionUid];
    state.pendingFiles.delete(filepath);
    
    // 发现完成且无待处理文件时，标记加载完成
    if (!state.isDiscovering && state.pendingFiles.size === 0 && state.isProcessing) {
      state.isProcessing = false;
      win.webContents.send('main:collection-loading-state-updated', {
        collectionUid,
        isLoading: false
      });
    }
  }

  // 完成发现阶段
  completeCollectionDiscovery(win, collectionUid) {
    if (!this.loadingStates[collectionUid]) return;
    
    const state = this.loadingStates[collectionUid];
    state.isDiscovering = false;
    
    // 有待处理文件，进入处理阶段
    if (state.pendingFiles.size > 0) {
      state.isProcessing = true;
    } else {
      // 无待处理文件，直接标记加载完成
      win.webContents.send('main:collection-loading-state-updated', {
        collectionUid,
        isLoading: false
      });
    }
  }
}
```

### 4.2 状态流转图

```
startCollectionDiscovery
        ↓
isDiscovering = true, pendingFiles.clear()
        ↓
    发现文件 → addFileToProcessing → pendingFiles.add(file)
        ↓
    onWatcherSetupComplete (ready 事件)
        ↓
completeCollectionDiscovery
        ↓
isDiscovering = false
        ↓
  ┌───────────────────────────────┐
  │ pendingFiles.size > 0 ?       │
  ├───────────┬───────────────────┤
  │ Yes       │ No                │
  │ ↓         │ ↓                 │
  │ isProcessing = true           │
  │           │ isLoading = false │
  │           │ (发送到 UI)       │
  │ ↓         │                   │
  │ 处理文件                     │
  │ markFileAsProcessed           │
  │ pendingFiles.delete(file)     │
  │ ↓                             │
  │ pendingFiles.size === 0 ?     │
  │ ↓                             │
  │ isProcessing = false          │
  │ isLoading = false             │
  │ (发送到 UI)                   │
  └───────────────────────────────┘
```

## 5. 去重加载层 (Deduplication)

### 5.1 文件去重机制

```javascript
collectionAddFileEvent: (state, action) => {
  const file = action.payload.file;
  const collection = findCollectionByUid(state.collections, file.meta.collectionUid);
  
  if (collection) {
    // 查找是否已存在相同 UID 的项
    const currentItem = find(currentSubItems, (i) => i.uid === file.data.uid);
    
    if (currentItem) {
      // 已存在，执行更新而非添加
      currentItem.name = file.data.name;
      currentItem.type = file.data.type;
      currentItem.seq = file.data.seq;
      currentItem.request = mergeRequestWithPreservedUids(currentItem.request, file.data.request);
      currentItem.draft = null;  // 清除草稿（外部文件变更）
    } else {
      // 不存在，添加新项
      currentSubItems.push({
        uid: file.data.uid,
        name: file.data.name,
        type: file.data.type,
        // ... 其他属性
      });
    }
  }
}
```

### 5.2 重命名场景处理

文件重命名时，操作系统会触发 `unlink`（旧文件）和 `add`（新文件）两个事件，顺序不确定。通过 UID 去重机制确保不会出现重复项：

1. 如果 `add` 先触发：通过 UID 找到旧项，执行更新
2. 如果 `unlink` 先触发：删除旧项，`add` 时创建新项（通过 100ms 延迟减少此情况）

### 5.3 部分加载与完整加载

对于大文件，实现了分阶段加载避免 UI 阻塞：

```javascript
// collection-watcher.js add 函数
const fileStats = fs.statSync(pathname);

if (fileStats.size < MAX_FILE_SIZE) { // MAX_FILE_SIZE = 2.5MB
  // 小文件：直接解析并发送完整数据
  file.data = await parseRequestViaWorker(content, { format, filename: pathname });
  file.partial = false;
  file.loading = false;
} else {
  // 大文件：先发送元数据，标记为 partial
  file.data = parseFileMeta(content, format);
  file.partial = true;
  file.loading = false;
}
```

---

## 6. 集合重复打开与路径去重：避免重复加载的完整流程

### 6.1 四层防御机制概览

集合重复打开检测采用**四层防御架构**，从主进程到渲染进程形成完整的去重链条：

```
┌─────────────────────────────────────────────────────────────┐
│  第 1 层：用户交互层（对话框选择去重）                       │
│  openCollectionDialog: [...new Set(filePaths)]              │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│  第 2 层：主进程 API 层（路径规范化去重）                     │
│  openCollectionsByPathname: seenPaths Set + path.normalize │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│  第 3 层：watcher 管理层（watcher 实例去重）                  │
│  CollectionWatcher.hasWatcher() + watchers{} 映射表         │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│  第 4 层：渲染进程状态层（Redux 状态去重）                   │
│  openCollectionEvent: existingCollection + isAlreadyInWorkspace │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 第 1 层：用户交互层 - 对话框选择去重

**文件**: `packages/bruno-electron/src/app/collections.js`

用户通过文件对话框选择多个目录时，第一层去重立即生效：

```javascript
const openCollectionDialog = async (win, watcher) => {
  const { canceled, filePaths } = await dialog.showOpenDialog(win, {
    properties: ['openDirectory', 'createDirectory', 'multiSelections']
  });

  if (!canceled && filePaths?.length > 0) {
    // 第一层去重：使用 Set 去除用户重复选择的相同路径
    // 例如用户在对话框中不小心选择了同一目录两次
    const { openCollectionPromises, invalidPaths } = [...new Set(filePaths)].reduce(
      (acc, filePath) => {
        const resolvedPath = path.resolve(filePath);

        if (isDirectory(resolvedPath)) {
          acc.openCollectionPromises.push(
            openCollection(win, watcher, resolvedPath).catch(...)
          );
        } else {
          acc.invalidPaths.push(resolvedPath);
        }
        return acc;
      },
      { openCollectionPromises: [], invalidPaths: [] }
    );

    await Promise.all(openCollectionPromises);
  }
};
```

**作用**: 防止用户在单次选择中重复选择同一目录。

### 6.3 第 2 层：主进程 API 层 - 路径规范化去重

**文件**: `packages/bruno-electron/src/app/collections.js`

批量打开集合时，第二层去重处理不同形式的相同路径（如相对路径/绝对路径、大小写差异等）：

```javascript
const openCollectionsByPathname = async (win, watcher, collectionPaths, options = {}) => {
  const seenPaths = new Set();  // 跟踪已处理的规范化路径
  const result = { opened: [], failed: [], invalid: [] };

  for (const collectionPath of collectionPaths) {
    // 1. 路径解析与规范化
    const resolvedPath = path.isAbsolute(collectionPath)
      ? collectionPath
      : normalizeAndResolvePath(collectionPath);  // 相对路径转绝对路径
    
    const normalizedPath = path.normalize(resolvedPath);  // 统一路径分隔符、解析 . 和 ..
    
    // 2. 去重检查
    if (seenPaths.has(normalizedPath)) {
      continue;  // 跳过已处理的路径
    }
    seenPaths.add(normalizedPath);
    
    // 3. 调用 openCollection（进入第三层检查）
    if (isDirectory(resolvedPath)) {
      const openResult = await openCollection(win, watcher, resolvedPath, options);
      // ...
    }
  }
};
```

**作用**: 
- 将相对路径、符号链接、大小写差异的路径归一化
- 防止批量导入时同一物理路径被多次处理

### 6.4 第 3 层：watcher 管理层 - watcher 实例去重

**文件**: `packages/bruno-electron/src/app/collection-watcher.js`

这是**最关键的去重层**，直接防止创建重复的文件系统 watcher：

```javascript
class CollectionWatcher {
  constructor() {
    this.watchers = {};  // 核心映射表：key = 规范化路径, value = watcher 实例
    this.loadingStates = {};
    this.tempDirectoryMap = {};
  }

  /**
   * 检查路径是否已有 watcher
   * 注意：实际实现中 watchPath 已经是规范化的
   */
  hasWatcher(watchPath) {
    return this.watchers[watchPath];  // 直接查表
  }

  /**
   * 添加 watcher，先检查是否已存在
   */
  addWatcher(win, watchPath, collectionUid, brunoConfig, forcePolling = false, useWorkerThread) {
    // 防御性检查：如果已存在，先关闭旧的（实际场景中 hasWatcher 应已提前检查）
    if (this.watchers[watchPath]) {
      this.watchers[watchPath].close();
    }

    this.initializeLoadingState(collectionUid);
    this.startCollectionDiscovery(win, collectionUid);

    // 延迟 100ms 创建 watcher，避免快速连续调用导致的竞态条件
    setTimeout(() => {
      const watcher = chokidar.watch(watchPath, {
        ignoreInitial: false,
        usePolling: isWSLPath(watchPath) || forcePolling,
        awaitWriteFinish: {
          stabilityThreshold: 80,
          pollInterval: 10
        }
      });

      // 注册事件监听
      watcher
        .on('ready', () => onWatcherSetupComplete(win, watchPath, collectionUid, this))
        .on('add', (pathname) => add(...))
        .on('change', (pathname) => change(...));

      this.watchers[watchPath] = watcher;  // 存入映射表
    }, 100);
  }

  removeWatcher(watchPath, win, collectionUid) {
    if (this.watchers[watchPath]) {
      this.watchers[watchPath].close();
      this.watchers[watchPath] = null;  // 置为 null 而非 delete，保留键存在性检测
    }
  }
}
```

**关键设计**:
- `watchers` 对象以**规范化路径**为 key，确保同一物理目录只会有一个 watcher 实例
- `hasWatcher()` 是同步查询，在 `openCollection` 中被调用，作为快速检查点

### 6.5 第 4 层：渲染进程状态层 - Redux 状态去重

**文件**: `packages/bruno-app/src/providers/ReduxStore/slices/collections/actions.js`

即使前面三层都通过了，渲染进程在接收 `collection-opened` 事件时还会做最后一次检查：

```javascript
export const openCollectionEvent = (uid, pathname, brunoConfig) => (dispatch, getState) => {
  return new Promise((resolve, reject) => {
    const state = getState();
    const activeWorkspace = state.workspaces.workspaces.find(
      (w) => w.uid === state.workspaces.activeWorkspaceUid
    );

    // 检查 1: 集合是否已在 Redux store 中
    const existingCollection = state.collections.collections.find(
      (c) => normalizePath(c.pathname) === normalizePath(pathname)
    );

    // 检查 2: 集合是否已在当前工作区中
    const isAlreadyInWorkspace = activeWorkspace?.collections?.some(
      (c) => normalizePath(c.path) === normalizePath(pathname)
    );

    // 如果两个检查都通过，说明集合已完全加载，直接返回
    if (existingCollection && isAlreadyInWorkspace) {
      toast.success('Collection is already opened');
      resolve();
      return;
    }

    // 如果集合已加载但不在当前工作区，添加到工作区即可
    if (existingCollection) {
      if (activeWorkspace) {
        const workspaceCollection = { name: brunoConfig.name, path: pathname };
        ipcRenderer.invoke(
          'renderer:add-collection-to-workspace',
          activeWorkspace.pathname,
          workspaceCollection
        );
      }

      // 恢复该集合的标签页（无需重新加载文件）
      ipcRenderer.invoke('renderer:snapshot:get')
        .then((snapshot) => hydrateSnapshotLookups(snapshot || {}))
        .then((snapshotLookups) => hydrateCollectionTabs(
          existingCollection, dispatch, restoreTabs, snapshotLookups, activeWorkspace?.pathname || null
        ));

      resolve();
      return;
    }

    // 否则执行完整的集合添加流程
    dispatch(createCollection({
      uid,
      name: brunoConfig.name,
      pathname,
      brunoConfig
    }));

    resolve();
  });
};
```

### 6.6 完整调用链路

```
用户选择目录
    ↓
[第1层] openCollectionDialog: Set 去重用户选择
    ↓
[第2层] openCollectionsByPathname: path.normalize + seenPaths Set
    ↓
    ↓─→ 重复路径：跳过
    │
    ↓─→ 新路径：调用 openCollection
              ↓
              [第3层] openCollection: watcher.hasWatcher() 检查
                        ↓
                        ↓─→ watcher 已存在：仅发送 collection-opened
                        │        ↓
                        │        [第4层] openCollectionEvent: existingCollection 检查
                        │                  ↓
                        │                  ↓─→ 已存在：跳过，显示提示
                        │                  ↓─→ 不存在：添加到 Redux
                        │
                        ↓─→ watcher 不存在：创建 watcher + 发送 collection-opened
                                  ↓
                                  [第4层] openCollectionEvent: 添加到 Redux store
```

### 6.7 去重机制有效性

| 场景 | 处理层级 | 处理方式 |
|------|---------|---------|
| 用户在对话框中重复选择同一目录 | 第 1 层 | `[...new Set(filePaths)]` 去重 |
| 工作区配置中有重复集合路径 | 第 2 层 | `seenPaths` Set + `path.normalize` |
| 同一目录通过符号链接/相对路径访问 | 第 2 层 | `normalizeAndResolvePath` 解析 |
| 用户尝试打开已加载的集合 | 第 3 层 | `watcher.hasWatcher()` 返回 true |
| 集合已加载但不在当前工作区 | 第 4 层 | `existingCollection` 存在，仅添加到工作区 |
| 集合已在当前工作区 | 第 4 层 | `isAlreadyInWorkspace` 为 true，直接返回 |

---

## 7. watcher ready 到 UI 快照恢复：调用链路与刷新时机

### 7.1 核心事件时序图

```
  主进程 (Main)                          渲染进程 (Renderer)
      │                                       │
      │  chokidar.watch() 创建                │
      │  ├─ 扫描文件系统                      │
      │  ├─ 触发 add/addDir 事件              │
      │  │    └─→ addFileToProcessing()       │
      │  │        └─→ pendingFiles.add()      │
      │                                       │
      │  ready 事件触发                       │
      │  └─→ onWatcherSetupComplete()         │
      │       ├─ completeCollectionDiscovery() │
      │       │   ├─ isDiscovering = false    │
      │       │   └─ isProcessing = true      │
      │       │                               │
      │       └─→ 发送 IPC 事件:              │
      │           main:hydrate-app-with-ui-state-snapshot
      │                                       │
      │  (后台继续处理 pendingFiles)          │
      │  ├─ markFileAsProcessed()             │
      │  └─→ pendingFiles 为空时:             │
      │      isLoading = false                │
      │                                       │
      │  集合配置解析完成后发送:              │
      │  main:collection-opened               │
      │                                       │
      │                                       │  接收 main:collection-opened
      │                                       │  └─→ openCollectionEvent()
      │                                       │       ├─ 检查是否已存在
      │                                       │       ├─ createCollection
      │                                       │       └─  finally:
      │                                       │           hydrateSnapshotForOpenedCollection()
      │                                       │               ├─ 恢复标签页
      │                                       │               ├─ （如为活动集合）挂载集合
      │                                       │               ├─ 恢复活动标签页
      │                                       │               └─ maybeCompleteSnapshotHydrationSession()
      │                                       │                   └─ pending 为空时:
      │                                       │                       setSnapshotReady(true)
```

### 7.2 watcher ready 事件处理

**文件**: `packages/bruno-electron/src/app/collection-watcher.js`

```javascript
const onWatcherSetupComplete = (win, watchPath, collectionUid, watcher) => {
  // 职责 1: 标记发现阶段完成，更新加载状态机
  watcher.completeCollectionDiscovery(win, collectionUid);

  // 职责 2: 读取该集合的快照并发送到渲染进程
  const collectionSnapshotState = snapshotManager.getCollection(watchPath);

  const hydratePayload = collectionSnapshotState
    ? {
        pathname: watchPath,
        environmentPath: collectionSnapshotState?.environment?.collection || '',
        selectedEnvironment: collectionSnapshotState?.selectedEnvironment || ''
      }
    : null;

  // 发送快照 hydration 事件（轻量状态：环境选择等）
  win.webContents.send('main:hydrate-app-with-ui-state-snapshot', hydratePayload);
};
```

### 7.3 加载状态机与刷新时机

**文件**: `packages/bruno-electron/src/app/collection-watcher.js`

```javascript
completeCollectionDiscovery(win, collectionUid) {
  if (!this.loadingStates[collectionUid]) return;
  
  const state = this.loadingStates[collectionUid];
  state.isDiscovering = false;  // 发现阶段结束
  
  if (state.pendingFiles.size > 0) {
    state.isProcessing = true;  // 进入处理阶段，继续后台解析
  } else {
    win.webContents.send('main:collection-loading-state-updated', {
      collectionUid,
      isLoading: false
    });
  }
}

markFileAsProcessed(win, collectionUid, filepath) {
  const state = this.loadingStates[collectionUid];
  state.pendingFiles.delete(filepath);
  
  // 发现已完成且无待处理文件时，才发送加载完成事件
  if (!state.isDiscovering && state.pendingFiles.size === 0 && state.isProcessing) {
    state.isProcessing = false;
    win.webContents.send('main:collection-loading-state-updated', {
      collectionUid,
      isLoading: false
    });
  }
}
```

**刷新时机控制**:

| 阶段 | 触发条件 | UI 行为 |
|------|---------|--------|
| 开始发现 | `startCollectionDiscovery` | 显示加载骨架屏 `isLoading: true` |
| 发现完成 | `ready` 事件 → `completeCollectionDiscovery` | 保持加载状态，文件树结构已完整 |
| 处理完成 | `pendingFiles.size === 0` | 隐藏加载骨架屏 `isLoading: false`，完整渲染文件树 |

### 7.4 渲染进程快照恢复链路

**文件 1**: `packages/bruno-app/src/providers/App/useIpcEvents.js`

```javascript
// 监听器 1: 集合打开完成（文件树已加载）
const removeOpenCollectionListener = ipcRenderer.on(
  'main:collection-opened',
  async (pathname, uid, brunoConfig) => {
    try {
      await dispatch(openCollectionEvent(uid, pathname, brunoConfig));
    } finally {
      // 增量快照恢复的入口点
      dispatch(hydrateSnapshotForOpenedCollection(pathname));
    }
  }
);
```

**文件 2**: `packages/bruno-app/src/providers/ReduxStore/slices/workspaces/actions.js`

`hydrateSnapshotForOpenedCollection` 是增量恢复的核心：

```javascript
export const hydrateSnapshotForOpenedCollection = (collectionPathname) => {
  return async (dispatch, getState) => {
    const state = getState();
    const snapshotHydration = state.app.snapshotHydration;

    // 前置检查 1: 是否在快照恢复会话中
    if (!snapshotHydration?.workspaceUid) return;

    // 前置检查 2: 工作区是否匹配
    if (state.workspaces.activeWorkspaceUid !== snapshotHydration.workspaceUid) {
      clearSnapshotHydrationTimeout();
      dispatch(clearSnapshotHydrationSession());
      return;
    }

    // 前置检查 3: 该集合是否需要恢复
    const normalizedCollectionPath = normalizePath(collectionPathname);
    const isPendingHydration = snapshotHydration.pendingCollectionPathnames.some(
      (pathname) => normalizePath(pathname) === normalizedCollectionPath
    );
    if (!isPendingHydration) return;

    // 前置检查 4: 集合是否已在 Redux 中
    const collection = state.collections.collections.find(
      (c) => c.pathname && normalizePath(c.pathname) === normalizedCollectionPath
    );
    if (!collection) return;

    // === 开始恢复 ===
    const activeWorkspacePathname = state.workspaces.workspaces.find(
      (w) => w.uid === snapshotHydration.workspaceUid
    )?.pathname || null;

    // 步骤 1: 恢复该集合的所有标签页
    await hydrateTabs([collection], dispatch, restoreTabs, null, activeWorkspacePathname);

    // 步骤 2: 如果是快照记录的活动集合，执行额外恢复
    if (
      snapshotHydration.activeCollectionPathname
      && normalizePath(snapshotHydration.activeCollectionPathname) === normalizedCollectionPath
    ) {
      dispatch(expandCollection(collection.uid));

      const needsMount = collection.mountStatus !== 'mounted' && collection.mountStatus !== 'mounting';
      if (needsMount) {
        await dispatch(mountCollection({
          collectionUid: collection.uid,
          collectionPathname: collection.pathname,
          brunoConfig: collection.brunoConfig,
          skipTabRestore: true,
          workspacePathname: activeWorkspacePathname
        }));
      }

      const activeTab = await getActiveTabFromSnapshot(
        collection.pathname, collection, null, activeWorkspacePathname
      );
      if (activeTab) dispatch(addTab(activeTab));
    }

    // 步骤 3: 标记该集合已恢复
    dispatch(markSnapshotCollectionHydrated({ pathname: collection.pathname }));
    
    // 步骤 4: 检查是否所有集合都已恢复
    maybeCompleteSnapshotHydrationSession(dispatch, getState);
  };
};
```

### 7.5 快照恢复完成检测

```javascript
const maybeCompleteSnapshotHydrationSession = (dispatch, getState) => {
  const state = getState();
  const snapshotHydration = state.app.snapshotHydration;

  if (!snapshotHydration?.workspaceUid) return false;
  if (state.workspaces.activeWorkspaceUid !== snapshotHydration.workspaceUid) {
    clearSnapshotHydrationTimeout();
    dispatch(clearSnapshotHydrationSession());
    return false;
  }
  if (snapshotHydration.pendingCollectionPathnames.length > 0) return false;

  // === 所有集合恢复完成 ===
  clearSnapshotHydrationTimeout();
  dispatch(setSnapshotReady(true));
  dispatch(clearSnapshotHydrationSession());
  return true;
};

// 5 分钟超时保护：避免无限等待
const SNAPSHOT_HYDRATION_LONG_STOP_GUARD_MS = 5 * 60 * 1000;
```

### 7.6 刷新时机的三个关键节点

| 节点 | 触发事件 | UI 状态变化 | 用户可交互性 |
|------|---------|------------|--------------|
| 节点 1: watcher ready | `onWatcherSetupComplete` | 集合树结构完整，仍显示加载状态 | 不可交互 |
| 节点 2: pendingFiles 清空 | `markFileAsProcessed` 检测到 `size === 0` | `isLoading: false`，完整渲染文件树 | 可浏览文件树 |
| 节点 3: 快照恢复完成 | `maybeCompleteSnapshotHydrationSession` | `snapshotReady: true`，标签页恢复 | 完全可交互 |

### 7.7 竞态条件处理

1. **工作区切换检测**: 每次恢复前检查 `activeWorkspaceUid` 是否匹配
2. **超时保护**: 5 分钟强制结束恢复会话
3. **幂等设计**: `hydrateSnapshotForOpenedCollection` 可安全多次调用
4. **路径规范化**: 所有路径比较都使用 `normalizePath`

---

## 8. 多层级防抖总结

| 层级 | 位置 | 机制 | 延迟/阈值 |
|------|------|------|-----------|
| **文件写入防抖** | chokidar 配置 | awaitWriteFinish | 80ms stabilityThreshold |
| **删除事件防抖** | useIpcEvents.js | setTimeout | 100ms |
| **状态合并防抖** | collectionChangeFileEvent | UID 保留 + 草稿保护 | 即时（无延迟） |
| **批量刷新防抖** | loadingStates | 状态机 + pendingFiles | 直到所有文件处理完成 |
| **集合路径去重** | openCollection / loadWorkspaceCollectionsForSwitch | Set + Map + path.normalize | 即时（无延迟） |
| **快照恢复防抖** | hydrateSnapshotForOpenedCollection / maybeCompleteSnapshotHydrationSession | pendingCollectionPathnames + 5 分钟超时 | 直到所有集合恢复完成 |

## 9. 关键设计原则

1. **渐进式加载**：先展示文件树结构，再异步加载内容，提升感知速度
2. **UID 稳定性**：通过保留 UID 避免 React 不必要的重新渲染
3. **草稿保护**：外部文件变更时，仅当草稿与文件内容完全一致时才清除，保护用户编辑
4. **状态机控制**：通过 isDiscovering/isProcessing 状态标志，避免初始扫描时的多次 UI 刷新
5. **分层防抖**：从文件系统到 UI 渲染，每一层都有对应的防抖策略
6. **路径规范化**：在所有入口点统一使用 `path.normalize`，确保同一物理路径不会被重复处理
7. **快照恢复时机**：在 watcher ready 且集合结构完整后再恢复 UI 状态，避免部分加载时的状态错乱
8. **超时保护**：快照恢复设置 5 分钟超时，避免因个别集合加载失败导致整个工作区无法使用
