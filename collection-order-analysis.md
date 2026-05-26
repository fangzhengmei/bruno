# Collection 条目排序与改名副作用分析

## 概述

Bruno 使用文件系统存储 collection 数据，每个条目（文件夹/请求）的排序信息和名称都持久化在各自的文件中。排序和改名操作会触发多层面板的连锁更新，包括侧边栏显示、标签页引用、运行队列顺序等。

---

## 1. 排序持久化机制

### 1.1 核心数据模型

每个条目都有 `seq` 字段存储排序优先级：
- **文件夹**：存储在 `folder.bru` / `folder.yml` 的 `meta.seq` 字段
- **请求**：存储在 `*.bru` / `*.yml` 文件根层级的 `seq` 字段

### 1.2 排序计算流程

```
拖拽事件触发
    ↓
handleCollectionItemDrop (actions.js:1108)
    ├─ 同位置重排序 → handleReorderInSameLocation
    └─ 跨位置移动 → handleMoveToNewLocation
           ├─ 源目录：getReorderedItemsInSourceDirectory
           └─ 目标目录：getReorderedItemsInTargetDirectory
                    ↓
           updateItemsSequences (actions.js:1258)
                    ↓
           renderer:resequence-items (collection.js:1382)
                    ↓
           写入每个条目的文件系统
```

### 1.3 关键排序函数

#### `resetSequencesInFolder` [collections/index.js:1432-1439]
```javascript
// 先按名称+序号排序，然后重新分配连续的 seq 值
const sortedItems = sortByNameThenSequence(items);
return sortedItems.map((item, index) => {
  item.seq = index + 1;  // 重新分配为 1,2,3...
  return item;
});
```

#### `getReorderedItemsInTargetDirectory` [collections/index.js:1455-1476]
```javascript
// 1. 先重置所有序号为连续值
const itemsWithFixedSequences = resetSequencesInFolder(cloneDeep(items));
// 2. 标记拖拽条目和受影响条目
itemsWithFixedSequences.forEach((item) => {
  const isBetween = isItemBetweenSequences(item.seq, draggedSequence, targetSequence);
  if (isBetween) {
    item.seq += targetSequence > draggedSequence ? -1 : 1;
  }
  // 拖拽条目本身移到目标位置
  item.seq = calculateNewSequence(isDraggedItem, targetSequence, draggedSequence);
});
```

### 1.4 加载时排序算法 `sortByNameThenSequence`

**位置**：`packages/bruno-electron/src/utils/collection.js:855-894`

```javascript
// 1. 先按名称字母排序
const alphabeticallySorted = [...items].sort((a, b) => 
  a.name && b.name && a.name.localeCompare(b.name)
);

// 2. 分离有 seq 和无 seq 的条目
const withoutSeq = alphabeticallySorted.filter(f => !isSeqValid(f.seq));
const withSeq = alphabeticallySorted.filter(f => isSeqValid(f.seq))
                                    .sort((a, b) => a.seq - b.seq);

// 3. 将有 seq 的条目插入到指定位置（优先级高）
withSeq.forEach((item) => {
  const position = item.seq - 1;
  withoutSeq.splice(position, 0, item);
});

// 4. 处理 seq 冲突（相同位置的条目放在一起）
// 最后返回扁平化数组
return sortedItems.flat();
```

> **重要**：有 `seq` 的条目优先级高于字母排序。没有 `seq` 的条目按字母顺序排列。

### 1.5 持久化写入 `resequence-items`

**位置**：`packages/bruno-electron/src/ipc/collection.js:1382-1424`

```javascript
for (let item of itemsToResequence) {
  if (item.type === 'folder') {
    // 更新 folder.bru/yml 中的 meta.seq
    folderJsonData.meta.seq = item.seq;
    await writeFile(folderRootPath, content);
  } else if (REQUEST_TYPES.includes(item.type)) {
    // 更新请求文件中的 seq 字段
    const itemToSave = transformRequestToSaveToFilesystem(item);
    await writeFile(item.pathname, content);
  }
}
```

---

## 2. 改名引用更新机制

### 2.1 两种改名操作

| 操作 | IPC 通道 | 影响范围 |
|------|---------|---------|
| 仅改显示名 | `renderer:rename-item-name` | 仅文件内 `name` 字段 |
| 改文件名 | `renderer:rename-item-filename` | 文件路径 + name 字段 + 所有引用 |

### 2.2 仅改显示名 [collection.js:839-880]

```javascript
// 文件夹：更新 folder.bru 中的 meta.name
folderFileJsonContent.meta.name = newName;

// 请求：更新文件中的 name 字段
jsonData.name = newName;
```

- **不改变** `pathname`
- **不影响** `requestUids` 缓存
- **影响**：侧边栏显示、Tab 标题、运行时显示

### 2.3 改文件名 [collection.js:882-979]

#### 核心步骤：
1. 更新文件内的 `name` 字段
2. 重命名文件/文件夹（改变 `pathname`）
3. **更新 `requestUids` 缓存映射**
4. 文件夹改名时递归更新所有子项

#### 关键引用更新：

##### `requestUids` 缓存 [requestUids.js:1-81]
```javascript
const requestUids = new Map(); // pathname → uid

moveRequestUid(oldPathname, newPathname) {
  const uid = requestUids.get(oldPathname);
  if (uid) {
    requestUids.delete(oldPathname);
    requestUids.set(newPathname, uid);
  }
}
```

> **设计意图**：确保即使请求移动位置，其 `uid` 保持不变，避免丢失草稿状态。

##### 文件夹改名的递归处理 [collection.js:916-921]
```javascript
const requestFilesAtSource = await searchForRequestFiles(oldPath, collectionPathname);
for (let requestFile of requestFilesAtSource) {
  const newRequestFilePath = requestFile.replace(oldPath, newPath);
  moveRequestUid(requestFile, newRequestFilePath);  // 每个子项都要更新
}
```

### 2.4 改名对标签页（Tabs）的影响

- **同集合内改名**：Tab 保持打开，因为 `itemUid` 不变
- **跨集合移动**：Tab 被关闭 [actions.js:1246]
  ```javascript
  if (isCrossCollectionMove) {
    dispatch(closeTabs({ tabUids: [draggedItemUid] }));
  }
  ```

---

## 3. 对运行队列的影响

### 3.1 运行队列构建流程

**位置**：`packages/bruno-electron/src/ipc/network/index.js:1324-1366`

```
run-collection-folder 触发
    ↓
┌─ 递归模式 (recursive=true) ─┐
│  sortFolder(folder)          │ 对整个文件夹树排序
│  getAllRequestsInFolderRecursively(sortedFolder)  按排序收集
└─────────────────────────────┘
    ↓
┌─ 非递归模式 (recursive=false) ─┐
│  收集当前文件夹下的直接请求      │
│  sortByNameThenSequence(requests)  按 seq 排序
└────────────────────────────────┘
    ↓
标签过滤 (tags) → 运行时过滤
    ↓
选择过滤 (selectedRequestUids) → 最高优先级
```

### 3.2 `sortFolder` 递归排序

**位置**：`packages/bruno-electron/src/utils/collection.js:712-727`

```javascript
const sortFolder = (folder) => {
  // 分离文件夹和请求
  let folderItems = filter(items, item => item.type === 'folder');
  let requestItems = filter(items, item => item.type !== 'folder');
  
  // 文件夹：sortByNameThenSequence（字母+seq）
  folderItems = sortByNameThenSequence(folderItems);
  // 请求：仅按 seq 升序
  requestItems = requestItems.sort((a, b) => a.seq - b.seq);
  
  // 合并规则：文件夹在前，请求在后
  folder.items = folderItems.concat(requestItems);
  
  // 递归处理子文件夹
  each(folderItems, item => sortFolder(item));
};
```

### 3.3 执行顺序优先级

```
1. 最高优先级：selectedRequestUids（用户手动选择的运行顺序）
   [network/index.js:1352-1366]
   └─ 按用户在"Configure requests to run"中选择的顺序执行
   
2. 次优先级：seq 字段（拖拽排序）
   └─ 有 seq 的条目优先按 seq 排序
   
3. 默认优先级：字母排序
   └─ 无 seq 的条目按 name 字母顺序排列
```

### 3.4 运行时排序代码

```javascript
// 有手动选择时，覆盖默认排序
if (selectedRequestUids && selectedRequestUids.length > 0) {
  const uidIndexMap = new Map();
  selectedRequestUids.forEach((uid, index) => {
    uidIndexMap.set(uid, index);
  });
  
  folderRequests = folderRequests
    .filter(request => uidIndexMap.has(request.uid))
    .sort((a, b) => uidIndexMap.get(a.uid) - uidIndexMap.get(b.uid));
}
```

---

## 4. 副作用与互相牵动关系

### 4.1 排序操作的连锁影响

| 操作 | 触发时机 | 影响面板 | 持久化范围 |
|------|---------|---------|-----------|
| 拖拽排序 | 同目录内调整顺序 | 侧边栏、运行队列 | 拖拽条目+受影响条目（可能多个） |
| 删除条目 | 删除后重新计算 | 侧边栏、运行队列 | 源目录所有剩余条目 |
| 跨目录移动 | 改变父节点 | 源+目标目录的侧边栏和运行队列 | 源目录和目标目录的相关条目 |

**关键代码点**：
- 删除后重排：`actions.js:1074-1084`
- 移动后重排：`actions.js:1158-1190`

### 4.2 改名操作的连锁影响

| 改名类型 | 影响的面板/模块 | 需要更新的引用 |
|---------|----------------|---------------|
| 仅改显示名 | 侧边栏、Tab 标题、运行时显示 | 无（仅文件内 name 字段） |
| 改文件名 | 侧边栏、Tabs、运行队列、缓存 | pathname、requestUids 缓存、所有子项路径 |

### 4.3 多面板同步的关键时序

```
用户拖拽排序
    ↓
await dispatch(handleCollectionItemDrop(...))
    ├─ 更新内存中的 Redux state
    ├─ await updateItemsSequences(...)  // 写入文件系统
    └─ 文件监视器触发 collection-watcher
          └─ 重新加载 collection 数据
                └─ 刷新侧边栏 + 下一次运行使用新顺序
```

> **保证**：使用 `await` 确保文件系统写入完成后才 resolve，避免竞态条件。

### 4.4 潜在问题与设计权衡

#### 问题 1：seq 冲突处理
`sortByNameThenSequence` 中如果两个条目有相同的 `seq`，会被放在一起形成数组后 `flat()` 展开。这可能导致排序顺序不稳定，依赖于字母排序的次级顺序。

#### 问题 2：运行中修改的竞态
运行队列**每次运行时重新构建**，不做缓存。但如果运行过程中修改排序：
- 当前正在执行的队列不受影响
- 脚本中 `bru.runRequest()` 通过路径查找可能失败（如果目标被移动/改名）

#### 问题 3：跨集合 UID 保持
`requestUids` 缓存确保请求移动后 UID 不变，但跨集合移动时 Tab 被强制关闭，避免了跨集合的状态混淆。

---

## 5. 代码溯源索引

### 5.1 排序持久化

| 模块 | 文件位置 | 行号范围 |
|------|---------|---------|
| 排序计算 | `packages/bruno-app/src/utils/collections/index.js` | 1430-1500 |
| 拖拽处理 | `packages/bruno-app/src/providers/ReduxStore/slices/collections/actions.js` | 1108-1273 |
| 持久化写入 | `packages/bruno-electron/src/ipc/collection.js` | 1382-1424 |
| 加载排序 | `packages/bruno-electron/src/utils/collection.js` | 855-894 |
| 文件夹排序 | `packages/bruno-electron/src/utils/collection.js` | 712-727 |

### 5.2 改名引用更新

| 模块 | 文件位置 | 行号范围 |
|------|---------|---------|
| 改名 Action | `packages/bruno-app/src/providers/ReduxStore/slices/collections/actions.js` | 804-864 |
| 改显示名 IPC | `packages/bruno-electron/src/ipc/collection.js` | 838-880 |
| 改文件名 IPC | `packages/bruno-electron/src/ipc/collection.js` | 882-979 |
| UID 缓存 | `packages/bruno-electron/src/cache/requestUids.js` | 1-81 |

### 5.3 运行队列

| 模块 | 文件位置 | 行号范围 |
|------|---------|---------|
| 队列构建 | `packages/bruno-electron/src/ipc/network/index.js` | 1324-1366 |
| 递归收集 | `packages/bruno-electron/src/utils/collection.js` | 729-747 |
| 执行循环 | `packages/bruno-electron/src/ipc/network/index.js` | 1368-1478 |
| 按路径运行请求 | `packages/bruno-electron/src/ipc/network/index.js` | 1296-1309 |

---

## 6. 总结

### 核心设计原则

1. **文件系统作为唯一真相源**：所有排序和名称信息都持久化在文件中，支持 Git 版本控制
2. **UID 不变性**：通过 `requestUids` 缓存保证请求移动/改名后身份不变
3. **延迟计算**：运行队列每次运行时重新构建，确保使用最新状态
4. **异步同步**：通过 `await` 和文件监视器保证多面板状态最终一致

### 副作用传播路径

```
排序/改名操作
    ├─ 内存状态更新 (Redux)
    ├─ 文件系统持久化
    │   ├─ seq 字段写入
    │   ├─ name 字段更新
    │   └─ pathname 变更 + UID 缓存更新
    ├─ 侧边栏刷新 (collection-watcher)
    ├─ Tab 状态更新 (如跨集合移动则关闭)
    └─ 运行队列 (下次运行时重新排序)
```

**关键洞察**：排序和改名不是简单的 UI 操作，而是会触发文件系统写入、缓存更新、多面板同步的全链路操作。理解 `seq` 字段的优先级和 `requestUids` 缓存的作用是排查相关问题的关键。
