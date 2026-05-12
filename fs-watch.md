# 文件系统监听与内存模型增量更新链路设计

## 概述

Bruno 桌面端采用 **chokidar** 作为文件系统监听器，实时监测 collection 目录下的文件变更。当外部编辑器修改文件时，系统通过事件去抖、解析和增量 patch 机制，实现侧栏树和打开标签的实时刷新，同时只更新必要的节点。

---

## 1. 监听器实现

### 1.1 核心依赖

使用 **chokidar** 作为文件监听库，它封装了 Node.js 原生的 `fs.watch` 和 `fs.watchFile`，并提供了跨平台的一致性保证。

**文件位置**: `packages/bruno-electron/src/app/collection-watcher.js`

### 1.2 监听器初始化

```javascript
addWatcher(win, watchPath, collectionUid, brunoConfig, forcePolling = false, useWorkerThread) {
  // 关闭旧的监听器
  if (this.watchers[watchPath]) {
    this.watchers[watchPath].close();
  }

  // 初始化加载状态
  this.initializeLoadingState(collectionUid);
  this.startCollectionDiscovery(win, collectionUid);

  // 默认忽略 node_modules 和 .git
  const defaultIgnores = ['node_modules', '.git'];
  const userIgnores = brunoConfig?.ignore || [];
  const ignores = [...new Set([...defaultIgnores, ...userIgnores])];

  setTimeout(() => {
    const watcher = chokidar.watch(watchPath, {
      ignoreInitial: false,
      usePolling: isWSLPath(watchPath) || forcePolling ? true : false,
      ignored: (filepath) => {
        // 忽略规则逻辑
        // ...
      },
      persistent: true,
      ignorePermissionErrors: true,
      awaitWriteFinish: {
        stabilityThreshold: 80,
        pollInterval: 10
      },
      depth: 20,
      disableGlobbing: true
    });

    // 绑定事件处理
    watcher
      .on('ready', () => onWatcherSetupComplete(win, watchPath, collectionUid, this))
      .on('add', (pathname) => add(win, pathname, collectionUid, watchPath, useWorkerThread, this))
      .on('addDir', (pathname) => addDirectory(win, pathname, collectionUid, watchPath))
      .on('change', (pathname) => change(win, pathname, collectionUid, watchPath))
      .on('unlink', (pathname) => unlink(win, pathname, collectionUid, watchPath))
      .on('unlinkDir', (pathname) => unlinkDir(win, pathname, collectionUid, watchPath))
      .on('error', (error) => { /* 错误处理 */ });

    this.watchers[watchPath] = watcher;
  }, 100);
}
```

### 1.3 关键配置说明

| 配置项 | 值 | 说明 |
|--------|-----|------|
| `ignoreInitial` | `false` | 初始扫描时触发 add 事件，用于首次加载整个目录树 |
| `usePolling` | 动态 | WSL 环境或强制轮询时使用，解决跨平台文件系统问题 |
| `ignored` | 函数 | 精确匹配忽略规则，包括 `.env` 文件、`node_modules`、`.git` 等 |
| `persistent` | `true` | 保持进程不退出 |
| `ignorePermissionErrors` | `true` | 忽略权限错误，避免崩溃 |
| `awaitWriteFinish` | 对象 | 等待文件写入完成后再触发事件（见事件去抖章节） |
| `depth` | `20` | 最大监听深度 |
| `disableGlobbing` | `true` | 禁用 glob 匹配，使用精确路径匹配 |

---

## 2. 事件去抖 (Debounce)

### 2.1 为什么需要去抖

文件保存时通常会触发多次 I/O 操作（创建临时文件、重命名、写入内容等），如果每次都立即处理会导致：
- 重复解析相同文件
- UI 频繁刷新产生闪烁
- 不必要的性能消耗

### 2.2 chokidar 内置去抖机制

```javascript
awaitWriteFinish: {
  stabilityThreshold: 80,  // 等待文件大小稳定 80ms
  pollInterval: 10         // 每 10ms 轮询一次文件状态
}
```

**工作原理**:
1. 文件首次变更时开始计时
2. 每 `pollInterval` (10ms) 检查一次文件大小和 mtime
3. 如果 `stabilityThreshold` (80ms) 内文件没有变化，才触发 `change` 事件
4. 有效过滤了编辑器保存时的多次写入操作

### 2.3 额外的去抖层

在 Redux 事件处理中，对于 `unlink` 事件还有额外的延迟处理：

**文件位置**: `packages/bruno-app/src/providers/App/useIpcEvents.js`

```javascript
if (type === 'unlink') {
  setTimeout(() => {
    dispatch(
      collectionUnlinkFileEvent({
        file: val
      })
    );
  }, 100);  // 额外 100ms 延迟
}
```

**设计意图**:
- 防止文件被快速删除后立即重建导致的状态不一致
- 给文件系统足够的时间完成删除操作
- 避免与编辑器的 "保存-删除-重建" 工作流冲突

---

## 3. 模型增量 Patch

### 3.1 事件数据流

```
文件系统变更
    ↓
chokidar 监听器 (主进程)
    ↓
add / change / unlink / addDir / unlinkDir 处理器
    ↓
解析文件内容 (parseRequest / parseCollection / parseFolder)
    ↓
IPC 发送到渲染进程: `main:collection-tree-updated`
    ↓
Redux Action 分发
    ↓
Reducer 执行增量更新
    ↓
React 组件重渲染 (侧栏树 + 打开的标签页)
```

### 3.2 文件类型识别与处理

在 `change` 函数中，根据文件路径和类型进行精确路由：

```javascript
const change = async (win, pathname, collectionUid, collectionPath) => {
  // 1. Collection 配置文件 (bruno.json)
  if (isBrunoConfigFile(pathname, collectionPath)) {
    // 更新配置 + 发送 IPC 事件
  }

  // 2. 环境变量文件 (environments/ 目录下)
  if (isEnvironmentsFolder(pathname, collectionPath)) {
    return changeEnvironmentFile(win, pathname, collectionUid, collectionPath);
  }

  // 3. Collection Root 文件 (collection.bru / opencollection.yml)
  if (isCollectionRootFile(pathname, collectionPath)) {
    // 解析并更新 root
  }

  // 4. Folder Root 文件 (folder.bru / folder.yml)
  if (isFolderRootFile(pathname, collectionPath)) {
    // 解析并更新 folder.root
  }

  // 5. 请求文件 (*.bru / *.yml)
  const format = getCollectionFormat(collectionPath);
  if (hasRequestExtension(pathname, format)) {
    // 解析请求文件并更新
  }
};
```

### 3.3 UID 保留机制 (核心优化)

**问题**: 文件重新解析后，如果重新生成 UID 会导致 React 认为是全新的节点，触发完整的卸载-重加载周期，包括：
- 失去编辑器光标位置
- 标签页状态重置
- 动画效果被打断

**解决方案**: `mergeRequestWithPreservedUids`

**文件位置**: `packages/bruno-app/src/providers/ReduxStore/slices/collections/index.js:109-113`

```javascript
/**
 * 保留现有数组项的 UID，合并新旧数据
 * 通过位置匹配 UID，保持 React keys 稳定
 */
const preserveUidsAtPaths = (existing, updated, paths) => {
  if (!existing || !updated) return updated;

  const merged = cloneDeep(updated);

  paths.forEach((path) => {
    const newArray = get(merged, path);
    const existingArray = get(existing, path, []);

    if (Array.isArray(newArray) && newArray.length) {
      set(
        merged,
        path,
        newArray.map((item, i) => 
          existingArray[i]?.uid 
            ? { ...item, uid: existingArray[i].uid } 
            : item
        )
      );
    }
  });

  return merged;
};

// 需要保留 UID 的路径配置
const REQUEST_UID_PATHS = [
  'params',
  'headers',
  'vars.req',
  'vars.res',
  'assertions',
  'body.formUrlEncoded',
  'body.multipartForm',
  'body.file'
];

const ROOT_UID_PATHS = ['request.headers', 'request.vars.req', 'request.vars.res'];

const mergeRequestWithPreservedUids = (existingRequest, newRequest) =>
  preserveUidsAtPaths(existingRequest, newRequest, REQUEST_UID_PATHS);

const mergeRootWithPreservedUids = (existingRoot, newRoot) =>
  preserveUidsAtPaths(existingRoot, newRoot, ROOT_UID_PATHS);
```

### 3.4 Reducer 增量更新实现

**collectionChangeFileEvent Reducer**:

```javascript
collectionChangeFileEvent: (state, action) => {
  const { file } = action.payload;
  const isCollectionRoot = file.meta.collectionRoot ? true : false;
  const isFolderRoot = file.meta.folderRoot ? true : false;
  const collection = findCollectionByUid(state.collections, file.meta.collectionUid);

  // Case 1: 更新 Collection Root
  if (isCollectionRoot) {
    if (collection) {
      collection.root = mergeRootWithPreservedUids(collection.root, file.data);
    }
    return;
  }

  // Case 2: 更新 Folder Root
  if (isFolderRoot) {
    const folderPath = path.dirname(file.meta.pathname);
    const folderItem = findItemInCollectionByPathname(collection, folderPath);
    if (folderItem) {
      folderItem.root = mergeRootWithPreservedUids(folderItem.root, file.data);
    }
    return;
  }

  // Case 3: 更新 Request 文件
  const item = findItemInCollectionByPathname(collection, file.meta.pathname);
  if (item) {
    // 保留现有 UID，只更新数据
    const mergedData = mergeRequestWithPreservedUids(item, file.data);
    
    // 增量更新 item，保持引用稳定性
    Object.assign(item, mergedData);
    
    // 同时更新文件元数据
    item.meta = {
      ...item.meta,
      ...file.meta,
      size: file.size
    };
  }
}
```

### 3.5 标签页同步机制

当打开的文件被外部修改时，通过 **Redux 状态订阅** 自动同步：

1. 标签页数据直接引用 Redux store 中的 collection items
2. Reducer 更新 item 时使用 `Object.assign` 保持对象引用
3. React Redux 通过 `shallowEqual` 检测到属性变化
4. 编辑器组件收到新 props 后重新渲染

**关键设计**:
- 不替换整个对象，只更新变化的属性
- 使用 `Object.assign` 而不是重新赋值
- UID 保持不变，React 不会重新挂载组件

---

## 4. 加载状态管理

### 4.1 Loading State 状态机

```javascript
initializeLoadingState(collectionUid) {
  if (!this.loadingStates[collectionUid]) {
    this.loadingStates[collectionUid] = {
      isDiscovering: false,    // 初始发现阶段
      isProcessing: false,     // 处理发现的文件
      pendingFiles: new Set()  // 待处理文件集合
    };
  }
}
```

### 4.2 状态流转

```
startCollectionDiscovery
    ↓
isDiscovering = true
pendingFiles.clear()
UI 显示加载中
    ↓
add 事件触发 → addFileToProcessing
pendingFiles.add(filepath)
    ↓
ready 事件触发
isDiscovering = false
    ↓
如果 pendingFiles 非空
    isProcessing = true
否则
    isProcessing = false
    UI 加载结束
    ↓
每个文件处理完成 → markFileAsProcessed
pendingFiles.delete(filepath)
    ↓
pendingFiles 为空 && !isDiscovering
    isProcessing = false
    UI 加载结束
```

---

## 5. 边界情况处理

### 5.1 大文件处理

```javascript
if (fileStats.size >= MAX_FILE_SIZE && format === 'bru') {
  file.data = await parseLargeRequestWithRedaction(content, 'bru');
  // 大文件精简解析，只保留必要元数据
}
```

### 5.2 Worker Thread 并行解析

```javascript
if (!useWorkerThread) {
  // 主线程直接解析
  file.data = await parseRequest(content, { format });
} else {
  // Worker 线程异步解析
  // 先发送部分信息到 UI，解析完成后再更新完整数据
  file.data = await parseRequestViaWorker(content, { format, filename: pathname });
}
```

### 5.3 错误恢复

```javascript
.on('error', (error) => {
  // ENOSPC: 文件监听器上限达到，降级为轮询模式
  if ((error.code === 'ENOSPC' || error.code === 'EMFILE') && !startedNewWatcher && !forcePolling) {
    startedNewWatcher = true;
    watcher.close();
    // 重试，使用 polling 模式
    this.addWatcher(win, watchPath, collectionUid, brunoConfig, true, useWorkerThread);
  }
});
```

---

## 6. 架构优势

### 6.1 精确更新
- 只更新被修改的单个文件节点
- 不重新加载整个 collection
- 不影响其他打开的标签页

### 6.2 状态稳定
- UID 保留机制确保 React 组件不被意外卸载
- 编辑器光标位置、滚动位置等状态得以保留
- 用户体验流畅

### 6.3 性能优化
- 事件去抖避免重复处理
- Worker 线程解析不阻塞 UI
- 大文件精简解析策略

### 6.4 容错设计
- 多级别错误处理
- 自动降级策略
- 权限错误忽略

---

## 总结

Bruno 的文件监听系统采用了 **三层架构**：

1. **底层**: chokidar 监听器 + awaitWriteFinish 去抖
2. **中间层**: 文件类型路由 + 解析策略选择
3. **上层**: Redux 增量 patch + UID 保留机制

这种设计确保了外部文件变更能够高效、准确地反映到 UI，同时保持应用状态的稳定性和用户体验的流畅性。
