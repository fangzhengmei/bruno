# Collection 运行顺序深度分析（R4 修正版）

> **修正说明**：本版本修正了 R3 中关于队列快照机制的不准确描述，明确了 IPC 结构化克隆导致的双重隔离、`folderRequests` 与闭包 `collection` 的对象引用关系，以及改名后旧/新名称和路径的实际行为差异。

---

## 1. 递归与非递归场景下的排序规则差异

### 1.1 代码入口对比

**位置**：`packages/bruno-electron/src/ipc/network/index.js:1324-1340`

```javascript
if (recursive) {
  let sortedFolder = sortFolder(folder);
  folderRequests = getAllRequestsInFolderRecursively(sortedFolder);
} else {
  each(folder.items, (item) => {
    if (item.request && !item.isTransient) {
      folderRequests.push(item);
    }
  });
  folderRequests = sortByNameThenSequence(folderRequests);
}
```

### 1.2 递归模式（recursive=true）

排序分**两层**，由 `sortFolder` 实现，位置：`packages/bruno-electron/src/utils/collection.js:712-727`

#### 第一层：同目录内排序

```javascript
const sortFolder = (folder = {}) => {
  const items = folder.items || [];
  let folderItems = filter(items, (item) => item.type === 'folder');
  let requestItems = filter(items, (item) => item.type !== 'folder');

  // 文件夹：sortByNameThenSequence（字母 + seq 混合排序）
  folderItems = sortByNameThenSequence(folderItems);
  // 请求：仅按 seq 升序排序（注意：不经过 sortByNameThenSequence！）
  requestItems = requestItems.sort((a, b) => a.seq - b.seq);

  // 合并规则：文件夹在前，请求在后
  folder.items = folderItems.concat(requestItems);

  // 递归处理子文件夹（就地修改原对象）
  each(folderItems, (item) => {
    sortFolder(item);
  });

  return folder;  // 返回原对象引用
};
```

**关键差异**：递归模式下，请求只按 `seq` 数字排序，**不做字母排序兜底**。如果请求没有 `seq` 字段，`a.seq - b.seq` 结果为 `NaN`，排序顺序不确定（取决于 JS 引擎的稳定性）。

#### 第二层：递归收集

`getAllRequestsInFolderRecursively` 按文件夹树的顺序**深度优先**收集请求：

```javascript
const getAllRequestsInFolderRecursively = (folder = {}) => {
  let requests = [];
  if (folder.items && folder.items.length) {
    folder.items.forEach((item) => {
      if (item.isTransient) return;
      if (item.type !== 'folder') {
        requests.push(item);  // 直接 push 对象引用
      } else {
        requests = requests.concat(getAllRequestsInFolderRecursively(item));
      }
    });
  }
  return requests;
};
```

**执行顺序**：同级中先文件夹后请求，进入子文件夹后递归收集子项，再回到父级处理下一个兄弟节点。

### 1.3 非递归模式（recursive=false）

只收集当前文件夹下的**直接请求**，不进入子文件夹：

```javascript
each(folder.items, (item) => {
  if (item.request && !item.isTransient) {
    folderRequests.push(item);  // 直接 push 对象引用
  }
});
folderRequests = sortByNameThenSequence(folderRequests);
```

**关键差异**：非递归模式使用 `sortByNameThenSequence`，对请求做**字母+seq 混合排序**，有 seq 优先生效，无 seq 按字母排序兜底。

### 1.4 排序规则对比表

| 维度 | 递归模式 | 非递归模式 |
|------|---------|-----------|
| **处理范围** | 整个文件夹树 | 仅当前目录 |
| **文件夹排序** | `sortByNameThenSequence`（字母+seq） | 不处理文件夹 |
| **请求排序** | 仅 `a.seq - b.seq` 数字排序 | `sortByNameThenSequence`（字母+seq） |
| **无 seq 兜底** | ❌ 无兜底，顺序不稳定 | ✅ 字母排序兜底 |
| **收集方式** | 深度优先遍历 | 直接 push |
| **子文件夹请求** | ✅ 包含 | ❌ 不包含 |
| **对象引用** | 与 `folder`/`collection` 同一对象树 | 与 `folder`/`collection` 同一对象树 |

### 1.5 `sortByNameThenSequence` 完整实现

**位置**：`packages/bruno-electron/src/utils/collection.js:855-894`

```javascript
const sortByNameThenSequence = (items = []) => {
  // Step 1: 按名称字母排序（localeCompare）
  const alphabeticallySorted = [...items].sort((a, b) => {
    if (a.name && b.name) {
      return a.name.localeCompare(b.name);
    }
    return 0;
  });

  // Step 2: 分离有 seq 和无 seq 的条目
  const withoutSeq = alphabeticallySorted.filter(f => !isSeqValid(f.seq));
  const withSeq = alphabeticallySorted
    .filter(f => isSeqValid(f.seq))
    .sort((a, b) => a.seq - b.seq);

  // Step 3: 将有 seq 的条目按 seq 值插入到指定位置
  withSeq.forEach((item) => {
    const position = item.seq - 1;
    withoutSeq.splice(position, 0, item);
  });

  // Step 4: 处理 seq 冲突（相同位置用数组分组后 flat 展开）
  // ... 去重冲突处理 ...
  return sortedItems.flat();
};
```

**优先级**：`seq` 位置 > 字母排序。有 `seq` 的条目会被 `splice` 到指定索引位置，可能覆盖字母排序的顺序。

---

## 2. 队列快照机制的双重隔离

### 2.1 完整对象传递链路

```
Redux store 中的 collection
    ↓ cloneDeep (actions.js:678)  —— 第 1 次隔离：渲染进程内深拷贝
collectionCopy（渲染进程内存）
    ↓ IPC.invoke（结构化克隆）  —— 第 2 次隔离：跨进程深拷贝
主进程: folder, collection （完全独立的对象树）
    ↓ sortFolder 就地修改 folder.items
    ↓ getAllRequestsInFolderRecursively 收集引用
folderRequests（数组元素指向主进程 collection 对象树）
    ↓ runRequestByItemPathname 闭包捕获 collection
findItemInCollectionByPathname(collection, pathname)
```

### 2.2 关键代码证据

#### 第 1 次隔离：渲染进程内 cloneDeep

**位置**：`packages/bruno-app/src/providers/ReduxStore/slices/collections/actions.js:678`

```javascript
let collectionCopy = cloneDeep(collection);  // 独立副本

// ... 添加全局环境变量 ...
collectionCopy.globalEnvironmentVariables = globalEnvironmentVariables;

const folder = findItemInCollection(collectionCopy, folderUid);  // 从副本中查找

// 通过 IPC 传递副本
ipcRenderer.invoke(
  'renderer:run-collection-folder',
  folder,          // 从 collectionCopy 中查找得到的引用
  collectionCopy,  // 克隆后的 collection
  ...
);
```

#### 第 2 次隔离：Electron IPC 结构化克隆

Electron 的 `ipcRenderer.invoke` 和 `ipcMain.handle` 使用 HTML 结构化克隆算法对参数进行序列化/反序列化。这意味着主进程收到的 `folder` 和 `collection` 是**全新的对象副本**，与渲染进程中的对象内存地址完全不同。

**位置**：`packages/bruno-electron/src/ipc/network/index.js:1274`

```javascript
ipcMain.handle(
  'renderer:run-collection-folder',
  async (event, folder, collection, environment, runtimeVariables, recursive, delay, tags, selectedRequestUids) => {
    // 这里的 folder 和 collection 是结构化克隆后的独立副本
    // 与渲染进程 Redux store 中的对象完全隔离
  }
);
```

### 2.3 `folderRequests` 与闭包 `collection` 的引用关系

```javascript
// 递归模式
let sortedFolder = sortFolder(folder);            // 就地修改 folder 对象
folderRequests = getAllRequestsInFolderRecursively(sortedFolder);
// folderRequests 数组是新的，但元素是 folder/collection 对象树中的直接引用

// 非递归模式
each(folder.items, (item) => {
  folderRequests.push(item);                       // 直接 push 对象引用
});

// 标签过滤（纯 filter，返回新数组，元素仍是原引用）
folderRequests = folderRequests.filter(...);

// 手动选择过滤+排序（filter + sort，返回新数组，元素仍是原引用）
folderRequests = folderRequests.filter(...).sort(...);

// 闭包中的 runRequestByItemPathname 使用同一个 collection
const runRequestByItemPathname = async (relativeItemPathname) => {
  // 使用同一个 collection 对象树查找
  const _item = cloneDeep(findItemInCollectionByPathname(collection, itemPathname));
};
```

**重要结论**：
- `folderRequests[i]` 和 `findItemInCollectionByPathname(collection, pathname)` 返回的对象是**同一个引用**
- 它们共享 `name`、`pathname`、`uid` 等属性
- 运行过程中对外部 collection 的任何修改（通过文件监视器）都不会影响这个闭包中的对象树

---

## 3. 标签过滤与手动选择的生效先后关系

### 3.1 完整处理流程

**位置**：`packages/bruno-electron/src/ipc/network/index.js:1325-1366`

```javascript
// ====== Step 1: 收集 + 排序（根据 recursive 模式）======
let folderRequests = [];
if (recursive) {
  let sortedFolder = sortFolder(folder);
  folderRequests = getAllRequestsInFolderRecursively(sortedFolder);
} else {
  each(folder.items, (item) => {
    if (item.request && !item.isTransient) {
      folderRequests.push(item);
    }
  });
  folderRequests = sortByNameThenSequence(folderRequests);
}

// ====== Step 2: 标签过滤（先过滤）======
if (tags && tags.include && tags.exclude) {
  const includeTags = tags.include ? tags.include : [];
  const excludeTags = tags.exclude ? tags.exclude : [];
  folderRequests = folderRequests.filter(({ tags: requestTags = [], draft }) => {
    requestTags = draft?.tags || requestTags || [];
    return isRequestTagsIncluded(requestTags, includeTags, excludeTags);
  });
}

// ====== Step 3: 手动选择过滤 + 重排序（后过滤，且覆盖排序）======
if (selectedRequestUids && selectedRequestUids.length > 0) {
  const uidIndexMap = new Map();
  selectedRequestUids.forEach((uid, index) => {
    uidIndexMap.set(uid, index);
  });

  folderRequests = folderRequests
    .filter((request) => uidIndexMap.has(request.uid))
    .sort((a, b) => {
      const indexA = uidIndexMap.get(a.uid);
      const indexB = uidIndexMap.get(b.uid);
      return indexA - indexB;
    });
}
```

### 3.2 执行顺序与覆盖关系

```
Step 1: 收集所有候选请求并排序
        ↓ 输出：按 seq/字母排序的完整请求列表
Step 2: 标签过滤 (filter)
        ↓ 输出：保留匹配标签的请求，顺序不变
Step 3: 手动选择过滤 (filter + sort)
        ↓ 输出：仅保留选中的请求，按用户选择顺序重新排序
```

### 3.3 覆盖关系详解

| 阶段 | 操作类型 | 对顺序的影响 | 对上一阶段的覆盖 |
|------|---------|-------------|-----------------|
| Step 1 | 排序 | 建立默认顺序 | 无 |
| Step 2 | 过滤 | **不改变顺序**，只移除不匹配项 | 不覆盖顺序，仅缩减集合 |
| Step 3 | 过滤 + 重排序 | **完全覆盖顺序**，按 `selectedRequestUids` 数组顺序排序 | **完全覆盖** Step 1 的排序 |

### 3.4 关键行为细节

**标签过滤不改变顺序**（纯 filter 操作）：
```javascript
folderRequests = folderRequests.filter(({ tags, draft }) => {
  requestTags = draft?.tags || tags || [];
  return isRequestTagsIncluded(requestTags, includeTags, excludeTags);
});
```

**手动选择完全覆盖顺序**（filter + 显式 sort）：
```javascript
const uidIndexMap = new Map();
selectedRequestUids.forEach((uid, index) => uidIndexMap.set(uid, index));

folderRequests = folderRequests
  .filter(request => uidIndexMap.has(request.uid))
  .sort((a, b) => uidIndexMap.get(a.uid) - uidIndexMap.get(b.uid));
```

> `uidIndexMap` 将 UID 映射到其在 `selectedRequestUids` 数组中的位置索引，`sort` 按索引升序排列，完全决定了最终执行顺序。

### 3.5 最终执行顺序优先级

```
1. 最高优先级：selectedRequestUids 顺序（用户手动勾选的顺序）
   └─ 完全覆盖 seq 和字母排序

2. 次优先级：标签过滤后的保留项
   └─ 保持 Step 1 的排序顺序不变

3. 基础排序：
   ├─ 递归模式：请求按 seq 升序，文件夹按 sortByNameThenSequence
   └─ 非递归模式：所有请求按 sortByNameThenSequence
```

---

## 4. 改名后按名称跳转执行的影响

### 4.1 跳转执行机制

跳转由 `bru.setNextRequest(name)` 或 `bru.runner.setNextRequest(name)` 触发，位置：`packages/bruno-js/src/bru.js:81-83` 和 `380-382`

```javascript
// 构造函数中
this.runner = {
  setNextRequest: (nextRequest) => {
    this.nextRequest = nextRequest;  // 存储下一个要执行的请求名
  }
};

// 直接方法
setNextRequest(nextRequest) {
  this.nextRequest = nextRequest;
}
```

### 4.2 跳转执行流程

**位置**：`packages/bruno-electron/src/ipc/network/index.js:1875-1892`

```javascript
if (nextRequestName !== undefined) {
  nJumps++;
  if (nJumps > 10000) {
    throw new Error('Too many jumps, possible infinite loop');
  }
  if (nextRequestName === null) {
    break;  // 终止执行
  }
  // 按 name 字段在 folderRequests 中查找（注意：使用 === 严格相等）
  const nextRequestIdx = folderRequests.findIndex(
    (request) => request.name === nextRequestName
  );
  if (nextRequestIdx >= 0) {
    currentRequestIndex = nextRequestIdx;  // 跳转到找到的请求
  } else {
    console.error('Could not find request with name \'' + nextRequestName + '\'');
    currentRequestIndex++;  // 找不到则顺序继续
  }
} else {
  currentRequestIndex++;  // 无跳转则顺序下一个
}
```

### 4.3 跳转查找的数据源

跳转使用的 `folderRequests` 是**运行开始时构建的快照**，其构建流程为：

```
运行开始 → 收集请求 → 标签过滤 → 手动选择过滤
                                    ↓
                         folderRequests 数组（元素指向 collection 对象树）
                                    ↓
                         bru.setNextRequest(name)
                                    ↓
                         findIndex(request.name === name) （在快照数组中查找）
```

### 4.4 改名操作的三层影响

改名操作会同时影响三个层面，但只有**前两层**会相互同步，**第三层**（运行时闭包）完全隔离：

| 层面 | 改名是否影响 | 同步方式 |
|------|-------------|---------|
| 1. 文件系统 | ✅ 影响 | 直接写入文件/重命名文件 |
| 2. 渲染进程 Redux store | ✅ 影响 | 文件监视器触发重新加载 |
| 3. 主进程运行时闭包 | ❌ 不影响 | IPC 结构化克隆后完全隔离 |

### 4.5 场景 A：仅改显示名（rename-item-name）

**操作**：更新文件内 `name` 字段，不改变 pathname。

**位置**：`packages/bruno-electron/src/ipc/collection.js:839-880`

```javascript
folderFileJsonContent.meta.name = newName;  // 改 folder.bru 中的 meta.name
jsonData.name = newName;                    // 改请求文件中的 name
```

| 操作 | 运行时行为 | 原因 |
|------|-----------|------|
| `bru.setNextRequest('旧名称')` | ✅ **成功** | `folderRequests` 快照中 `name` 仍是旧值 |
| `bru.setNextRequest('新名称')` | ❌ **失败** | 快照中 `name` 未更新，`findIndex` 返回 -1 |
| `bru.runRequest('旧路径')` | ✅ **成功** | `pathname` 未变，闭包 `collection` 中能找到 |
| `bru.runRequest('新路径')` | —— 不适用 | 仅改 `name` 不影响 `pathname` |

**失败时的行为**：`currentRequestIndex++` 顺序执行下一个请求，并在控制台打印错误日志。

### 4.6 场景 B：改文件名（rename-item-filename）

**操作**：重命名文件/文件夹（改变 `pathname`），同时更新文件内 `name` 字段。

**位置**：`packages/bruno-electron/src/ipc/collection.js:882-979`

```javascript
await renameFile(oldPath, newPath);  // 改变文件系统路径
moveRequestUid(oldPathname, newPathname);  // 更新 Redux store 中的 UID 缓存
```

| 操作 | 运行时行为 | 原因 |
|------|-----------|------|
| `bru.setNextRequest('旧名称')` | ✅ **成功** | `folderRequests` 快照中 `name` 仍是旧值 |
| `bru.setNextRequest('新名称')` | ❌ **失败** | 快照中 `name` 未更新 |
| `bru.runRequest('旧路径')` | ✅ **成功** | 闭包 `collection` 快照中 `pathname` 仍是旧值，找到对象后直接从内存执行，不依赖文件系统 |
| `bru.runRequest('新路径')` | ❌ **失败** | 快照中 `pathname` 未更新，`findItemInCollectionByPathname` 找不到 |

**关键修正**（与 R3 版本不同）：
- `runRequest` 函数**不依赖文件系统**读取请求内容，完全使用内存中的 `item` 对象
- 即使文件系统中该路径已不存在，只要闭包 `collection` 快照中有该对象，就能正常执行
- `pathname` 仅用于在内存对象树中查找，不用于文件系统读取

### 4.7 场景 C：跨文件夹移动 + 跳转

**操作**：将请求从文件夹 A 移动到文件夹 B（改变 `pathname`）。

| 操作 | 运行时行为 | 原因 |
|------|-----------|------|
| `bru.setNextRequest('请求名')` | ✅ **成功（如果仍在队列中）** | 如果运行模式是递归且包含新文件夹，且请求在 `folderRequests` 队列中 |
| `bru.setNextRequest('请求名')` | ❌ **失败（如果不在队列中）** | 如果运行模式是非递归，或新文件夹不在运行范围内 |
| `bru.runRequest('旧路径')` | ✅ **成功** | 闭包 `collection` 快照中 `pathname` 仍是旧值 |
| `bru.runRequest('新路径')` | ❌ **失败** | 快照中 `pathname` 未更新 |

### 4.8 场景 D：运行中多次改名 + 跳转

```
运行开始
    ↓
构建 folderRequests 快照：
  请求 A: name="login", pathname="/api/login.bru"
  请求 B: name="getUser", pathname="/api/user.bru"
    ↓
执行请求 A
  脚本中调用 bru.setNextRequest("login")  —— 跳转到请求 A（自身）
    ↓
用户在 UI 中将请求 A 改名为 "auth"，文件重命名为 "auth.bru"
    ↓
执行请求 B
  脚本中调用 bru.setNextRequest("auth")   —— ❌ 失败（快照中仍是 "login"）
  脚本中调用 bru.setNextRequest("login")  —— ✅ 成功（快照中仍是 "login"）
  脚本中调用 bru.runRequest("api/auth")   —— ❌ 失败（快照中仍是旧 pathname）
  脚本中调用 bru.runRequest("api/login")  —— ✅ 成功（快照中仍是旧 pathname）
```

### 4.9 按路径查找的内部机制

**位置**：`packages/bruno-electron/src/ipc/network/index.js:1296-1309`

```javascript
const runRequestByItemPathname = async (relativeItemPathname) => {
  return new Promise(async (resolve, reject) => {
    const format = getCollectionFormat(collection.pathname);
    let itemPathname = path.join(collection.pathname, relativeItemPathname);
    if (itemPathname && !hasRequestExtension(itemPathname, format)) {
      itemPathname = `${itemPathname}.${format}`;
    }
    // 注意：在闭包 collection 中查找，不是从文件系统读取
    const _item = cloneDeep(findItemInCollectionByPathname(collection, itemPathname));
    if (_item) {
      // 直接使用内存中的 _item 对象执行，不依赖文件系统
      const res = await runRequest({ item: _item, collection, ... });
      resolve(res);
    }
    reject(`bru.runRequest: invalid request path - ${itemPathname}`);
  });
};
```

**位置**：`packages/bruno-electron/src/utils/collection.js:606-610`

```javascript
const findItemInCollectionByPathname = (collection, pathname) => {
  let flattenedItems = flattenItems(collection.items);
  return findItemByPathname(flattenedItems, pathname);
};

const findItemByPathname = (items = [], pathname) => {
  return find(items, (i) => i.pathname === pathname);  // 严格相等比较
};
```

### 4.10 影响总结表

| 操作类型 | `bru.setNextRequest('旧名称')` | `bru.setNextRequest('新名称')` | `bru.runRequest('旧路径')` | `bru.runRequest('新路径')` |
|---------|-------------------------------|-------------------------------|---------------------------|---------------------------|
| 仅改显示名 | ✅ 成功 | ❌ 失败（顺序继续） | ✅ 成功 | —— 不适用 |
| 改文件名 | ✅ 成功 | ❌ 失败（顺序继续） | ✅ 成功（内存执行） | ❌ 失败 |
| 跨文件夹移动 | ✅ 成功（如在队列中） | ❌ 失败 | ✅ 成功（内存执行） | ❌ 失败 |

---

## 5. 完整运行队列构建与执行流程图

```
run-collection-folder 触发
    │
    ├─ 渲染进程：cloneDeep(collection) → collectionCopy
    │    └─ IPC.invoke 结构化克隆 → 主进程
    │
    ▼ 主进程闭包开始（完全隔离）
    │
    ▼
[Step 1] 收集请求
    ├─ recursive=true:
    │   sortFolder(folder)
    │   ├─ 文件夹: sortByNameThenSequence
    │   ├─ 请求: a.seq - b.seq（仅数字排序）
    │   └─ 深度优先遍历收集（引用同一对象树）
    │
    └─ recursive=false:
        收集当前目录请求（引用同一对象树）
        sortByNameThenSequence（字母+seq 混合排序）
    │
    ▼
[Step 2] 标签过滤
    filter(tags.include ∩ requestTags ∧ tags.exclude ∩ requestTags = ∅)
    └─ 不改变顺序，仅移除不匹配项
    │
    ▼
[Step 3] 手动选择过滤 + 重排序
    filter(selectedRequestUids 包含 request.uid)
    └─ sort(按 selectedRequestUids 数组索引)
    └─ 完全覆盖之前的排序
    │
    ▼
[Step 4] folderRequests 构建完成
    ├─ 数组是新创建的
    └─ 元素指向闭包 collection 对象树（同一引用）
    │
    ▼
[Step 5] 循环执行
    currentRequestIndex = 0
    while (currentRequestIndex < folderRequests.length)
    │
    ├─ const item = cloneDeep(folderRequests[currentRequestIndex])
    ├─ 执行当前请求（使用内存对象，不读文件系统）
    │
    ├─ 检查 nextRequestName
    │   ├─ undefined → currentRequestIndex++（顺序下一个）
    │   ├─ null → break（终止）
    │   └─ name 字符串 → folderRequests.findIndex(request.name === name)
    │       ├─ 找到 → currentRequestIndex = 找到的索引
    │       └─ 未找到 → currentRequestIndex++（顺序继续 + 控制台错误）
    │
    └─ 检查 stopExecution → break
    │
    ▼
[外部操作：改名/移动]
    ├─ 文件系统更新 ✅
    ├─ Redux store 更新 ✅（文件监视器）
    └─ 主进程闭包 collection ❌（完全隔离，不受影响）
```

---

## 6. 核心结论与之前版本的修正

### 6.1 R3 → R4 的关键修正

| 修正点 | R3 版本（不准确） | R4 版本（准确） |
|--------|-----------------|----------------|
| 隔离层级 | 仅提到 cloneDeep | 明确了**双重隔离**：渲染进程 cloneDeep + IPC 结构化克隆 |
| 对象引用关系 | 模糊说明快照 | 明确 `folderRequests` 元素与闭包 `collection` 是**同一对象树**的引用 |
| `runRequest` 路径查找 | 暗示可能从文件系统读取 | 明确**完全不依赖文件系统**，仅在内存对象树中按 `pathname` 查找 |
| 旧路径执行行为 | 未明确 | 即使文件系统中文件已被移动/改名，只要快照中有该对象就能正常执行 |
| 一致性保证 | 未明确 | `folderRequests[i].name` 和 `findItemInCollectionByPathname` 结果的属性**始终一致**，因为是同一引用 |

### 6.2 设计意图分析

1. **双重隔离**：确保运行过程中外部修改（改名、移动、删除）不会导致正在执行的队列崩溃或行为异常
2. **内存执行**：所有请求数据在运行开始时已加载到内存，避免运行中文件系统竞争
3. **引用一致**：`folderRequests` 和 `collection` 使用同一对象树，确保名称和路径查找的一致性
4. **优雅降级**：跳转失败时不会抛出异常终止运行，而是顺序执行下一个请求并打印错误日志

### 6.3 潜在问题与使用建议

1. **脚本中硬编码名称/路径**：如果脚本使用 `bru.setNextRequest('硬编码名称')` 或 `bru.runRequest('硬编码路径')`，改名后需要同步更新脚本
2. **运行中改名的用户体验**：用户在运行过程中改名，脚本跳转行为可能与预期不符（仍使用旧名称），但 UI 已显示新名称
3. **测试建议**：对于依赖跳转的工作流，建议使用相对路径或在运行前确保所有名称/路径稳定

---

## 7. 代码溯源索引

### 7.1 队列快照与隔离

| 模块 | 文件位置 | 行号 |
|------|---------|------|
| 渲染进程 cloneDeep | `packages/bruno-app/src/providers/ReduxStore/slices/collections/actions.js` | 678 |
| IPC 调用传递克隆副本 | `packages/bruno-app/src/providers/ReduxStore/slices/collections/actions.js` | 703-713 |
| 主进程 IPC handler | `packages/bruno-electron/src/ipc/network/index.js` | 1274 |

### 7.2 排序相关

| 模块 | 文件位置 | 行号 |
|------|---------|------|
| 运行队列入口 | `packages/bruno-electron/src/ipc/network/index.js` | 1324-1340 |
| `sortFolder` 递归排序 | `packages/bruno-electron/src/utils/collection.js` | 712-727 |
| `getAllRequestsInFolderRecursively` | `packages/bruno-electron/src/utils/collection.js` | 729-747 |
| `sortByNameThenSequence` | `packages/bruno-electron/src/utils/collection.js` | 855-894 |

### 7.3 过滤相关

| 模块 | 文件位置 | 行号 |
|------|---------|------|
| 标签过滤 | `packages/bruno-electron/src/ipc/network/index.js` | 1342-1350 |
| 手动选择过滤 | `packages/bruno-electron/src/ipc/network/index.js` | 1352-1366 |

### 7.4 跳转执行相关

| 模块 | 文件位置 | 行号 |
|------|---------|------|
| `bru.setNextRequest` | `packages/bruno-js/src/bru.js` | 380-382 |
| `bru.runner.setNextRequest` | `packages/bruno-js/src/bru.js` | 81-83 |
| 跳转逻辑（执行循环） | `packages/bruno-electron/src/ipc/network/index.js` | 1875-1892 |
| `nextRequestName` 检查 | `packages/bruno-electron/src/ipc/network/index.js` | 1507-1509 |
| `bru.runRequest` 按路径 | `packages/bruno-electron/src/ipc/network/index.js` | 1296-1309 |
| `findItemInCollectionByPathname` | `packages/bruno-electron/src/utils/collection.js` | 606-610 |
| `runRequest` 主函数 | `packages/bruno-electron/src/ipc/network/index.js` | 737 |
| `prepareRequest` | `packages/bruno-electron/src/ipc/network/prepare-request.js` | 354 |

### 7.5 改名相关

| 模块 | 文件位置 | 行号 |
|------|---------|------|
| 改显示名 IPC | `packages/bruno-electron/src/ipc/collection.js` | 838-880 |
| 改文件名 IPC | `packages/bruno-electron/src/ipc/collection.js` | 882-979 |
| `requestUids` 缓存 | `packages/bruno-electron/src/cache/requestUids.js` | 1-81 |
