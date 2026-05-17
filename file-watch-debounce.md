# Bruno 文件树监听防抖实现说明

## 1. 概述

Bruno 使用多层级的文件监听与防抖机制，确保文件系统变更能够准确、高效地反映到 UI 上，同时避免重复渲染和性能问题。整个实现分为四个核心层面：**变更感知层**、**事件合并层**、**刷新控制层**、**去重加载层**。

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

## 6. 多层级防抖总结

| 层级 | 位置 | 机制 | 延迟/阈值 |
|------|------|------|-----------|
| **文件写入防抖** | chokidar 配置 | awaitWriteFinish | 80ms stabilityThreshold |
| **删除事件防抖** | useIpcEvents.js | setTimeout | 100ms |
| **状态合并防抖** | collectionChangeFileEvent | UID 保留 + 草稿保护 | 即时（无延迟） |
| **批量刷新防抖** | loadingStates | 状态机 + pendingFiles | 直到所有文件处理完成 |

## 7. 关键设计原则

1. **渐进式加载**：先展示文件树结构，再异步加载内容，提升感知速度
2. **UID 稳定性**：通过保留 UID 避免 React 不必要的重新渲染
3. **草稿保护**：外部文件变更时，仅当草稿与文件内容完全一致时才清除，保护用户编辑
4. **状态机控制**：通过 isDiscovering/isProcessing 状态标志，避免初始扫描时的多次 UI 刷新
5. **分层防抖**：从文件系统到 UI 渲染，每一层都有对应的防抖策略
