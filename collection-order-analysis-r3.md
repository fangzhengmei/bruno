# Collection 运行顺序深度分析（R3 补全版）

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

  // 递归处理子文件夹
  each(folderItems, (item) => {
    sortFolder(item);
  });

  return folder;
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
        requests.push(item);
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
    folderRequests.push(item);
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
  const sortedItems = [];
  // ... 去重冲突处理 ...
  return sortedItems.flat();
};
```

**优先级**：`seq` 位置 > 字母排序。有 `seq` 的条目会被 `splice` 到指定索引位置，可能覆盖字母排序的顺序。

---

## 2. 标签过滤与手动选择的生效先后关系

### 2.1 完整处理流程

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

### 2.2 执行顺序与覆盖关系

```
Step 1: 收集所有候选请求并排序
        ↓ 输出：按 seq/字母排序的完整请求列表
Step 2: 标签过滤 (filter)
        ↓ 输出：保留匹配标签的请求，顺序不变
Step 3: 手动选择过滤 (filter + sort)
        ↓ 输出：仅保留选中的请求，按用户选择顺序重新排序
```

### 2.3 覆盖关系详解

| 阶段 | 操作类型 | 对顺序的影响 | 对上一阶段的覆盖 |
|------|---------|-------------|-----------------|
| Step 1 | 排序 | 建立默认顺序 | 无 |
| Step 2 | 过滤 | **不改变顺序**，只移除不匹配项 | 不覆盖顺序，仅缩减集合 |
| Step 3 | 过滤 + 重排序 | **完全覆盖顺序**，按 `selectedRequestUids` 数组顺序排序 | **完全覆盖** Step 1 的排序 |

### 2.4 关键行为细节

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

### 2.5 最终执行顺序优先级

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

## 3. 改名后按名称跳转执行的影响

### 3.1 跳转执行机制

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

### 3.2 跳转执行流程

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
  // 按 name 字段在已构建的请求列表中查找
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

### 3.3 跳转查找的数据源

跳转使用的 `folderRequests` 是**运行开始时构建的快照**，其构建流程为：

```
运行开始 → 收集请求 → 标签过滤 → 手动选择过滤
                                    ↓
                         folderRequests 快照（不可变）
                                    ↓
                         bru.setNextRequest(name)
                                    ↓
                         findIndex(request.name === name)
```

### 3.4 改名对跳转的影响

#### 场景 A：仅改显示名（rename-item-name）

```javascript
// 位置：packages/bruno-electron/src/ipc/collection.js:839-880
folderFileJsonContent.meta.name = newName;  // 改 folder.bru 中的 meta.name
jsonData.name = newName;                    // 改请求文件中的 name
```

| 影响点 | 结果 |
|--------|------|
| **运行中跳转** | ❌ **失败**。`folderRequests` 是运行开始时的快照，name 已被解析为旧值。改名后运行中的脚本调用 `bru.setNextRequest('新名称')` 找不到请求，会走 `currentRequestIndex++` 顺序继续。 |
| **下次运行跳转** | ✅ 正常。下次运行时会从文件系统重新加载 name 字段。 |
| **bru.runRequest(path)** | ✅ 正常。此 API 按 `pathname` 查找，不受 name 影响。 |

#### 场景 B：改文件名（rename-item-filename）

```javascript
// 位置：packages/bruno-electron/src/ipc/collection.js:882-979
await renameFile(oldPath, newPath);  // 改变 pathname
moveRequestUid(oldPathname, newPathname);  // 更新 UID 缓存
```

| 影响点 | 结果 |
|--------|------|
| **运行中跳转（按 name）** | ❌ **失败**。原因同场景 A。 |
| **运行中跳转（按 pathname）** | ❌ **失败**。`bru.runRequest()` 按 `pathname` 查找，但运行时使用的 collection 对象是运行开始时的快照，其 `items` 中的 `pathname` 仍是旧路径，`findItemInCollectionByPathname` 找不到新路径。 |
| **下次运行跳转（按 name）** | ✅ 正常。 |
| **下次运行跳转（按 pathname）** | ✅ 正常，但需要更新脚本中的路径字符串。 |

#### 场景 C：运行中改名 + 脚本跳转

```
运行开始 → 构建 folderRequests 快照（包含旧 name）
    ↓
请求 A 执行 → 脚本中调用 bru.setNextRequest('B_old_name')
    ↓
此时 B 已被改名为 'B_new_name'
    ↓
findIndex(request.name === 'B_old_name')
    ↓
结果：找不到（因为快照中的 name 已被更新？）
```

**关键点**：`folderRequests` 是浅拷贝，`request.name` 引用的是运行时内存中的对象。如果改名操作在运行过程中修改了 Redux state 并重新加载了 collection，而 `folderRequests` 是在运行开始时用 `cloneDeep` 克隆的独立副本，那么快照中的 name 不会被更新。

**位置**：`packages/bruno-electron/src/ipc/network/index.js:1380`

```javascript
const item = cloneDeep(folderRequests[currentRequestIndex]);
```

每次执行前对单个请求做 `cloneDeep`，但 `folderRequests` 数组本身也是 `cloneDeep` 自运行开始时的 collection 数据。文件监视器触发的 collection 重新加载不会影响正在运行的队列快照。

### 3.5 `bru.runRequest()` 按路径查找机制

**位置**：`packages/bruno-electron/src/ipc/network/index.js:1296-1309`

```javascript
const runRequestByItemPathname = async (relativeItemPathname) => {
  return new Promise(async (resolve, reject) => {
    const format = getCollectionFormat(collection.pathname);
    let itemPathname = path.join(collection.pathname, relativeItemPathname);
    if (itemPathname && !hasRequestExtension(itemPathname, format)) {
      itemPathname = `${itemPathname}.${format}`;
    }
    // 按 pathname 在 collection 中查找
    const _item = cloneDeep(findItemInCollectionByPathname(collection, itemPathname));
    if (_item) {
      const res = await runRequest({ item: _item, ... });
      resolve(res);
    }
    reject(`bru.runRequest: invalid request path - ${itemPathname}`);
  });
};
```

**使用的 collection 对象**：此函数使用的是运行开始时传入的 `collection` 参数，通过闭包捕获。如果运行中文件被改名：

- 旧 `pathname`：`findItemInCollectionByPathname` 在内存 collection 中找不到（因为文件监视器可能已重新加载），但如果 collection 是运行开始时的快照，则旧 pathname 仍可找到。
- 新 `pathname`：如果 collection 快照未更新，则新路径找不到。

### 3.6 影响总结表

| 操作 | 运行中按 name 跳转 | 运行中按 pathname 跳转 | 下次运行 |
|------|-------------------|----------------------|---------|
| 仅改显示名 | ❌ 失败（快照旧 name） | ✅ 正常（pathname 不变） | ✅ 正常 |
| 改文件名 | ❌ 失败（快照旧 name） | ❌ 失败（pathname 变更） | ✅ 正常（需更新脚本路径） |
| 跨文件夹移动 | ❌ 失败（快照中不在队列） | ❌ 失败（pathname 变更） | ✅ 正常 |

---

## 4. 完整运行队列构建与执行流程图

```
run-collection-folder 触发
    │
    ▼
[Step 1] 收集请求
    ├─ recursive=true:
    │   sortFolder(folder)
    │   ├─ 文件夹: sortByNameThenSequence
    │   ├─ 请求: a.seq - b.seq（仅数字排序）
    │   └─ 深度优先遍历收集
    │
    └─ recursive=false:
        收集当前目录请求
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
[Step 4] 构建 folderRequests 快照
    cloneDeep 后不可变，后续运行不重新加载
    │
    ▼
[Step 5] 循环执行
    currentRequestIndex = 0
    while (currentRequestIndex < folderRequests.length)
    │
    ├─ 执行当前请求
    │
    ├─ 检查 nextRequestName
    │   ├─ undefined → currentRequestIndex++（顺序下一个）
    │   ├─ null → break（终止）
    │   └─ name 字符串 → findIndex(name 匹配)
    │       ├─ 找到 → currentRequestIndex = 找到的索引
    │       └─ 未找到 → currentRequestIndex++（顺序继续）
    │
    └─ 检查 stopExecution → break
```

---

## 5. 代码溯源索引

### 5.1 排序相关

| 模块 | 文件位置 | 行号 |
|------|---------|------|
| 运行队列入口 | `packages/bruno-electron/src/ipc/network/index.js` | 1324-1340 |
| `sortFolder` 递归排序 | `packages/bruno-electron/src/utils/collection.js` | 712-727 |
| `getAllRequestsInFolderRecursively` | `packages/bruno-electron/src/utils/collection.js` | 729-747 |
| `sortByNameThenSequence` | `packages/bruno-electron/src/utils/collection.js` | 855-894 |
| `sortCollection` | `packages/bruno-electron/src/utils/collection.js` | 694-710 |

### 5.2 过滤相关

| 模块 | 文件位置 | 行号 |
|------|---------|------|
| 标签过滤 | `packages/bruno-electron/src/ipc/network/index.js` | 1342-1350 |
| 手动选择过滤 | `packages/bruno-electron/src/ipc/network/index.js` | 1352-1366 |
| `isRequestTagsIncluded` | `packages/bruno-electron/src/utils/collection.js` | （标签工具函数） |

### 5.3 跳转执行相关

| 模块 | 文件位置 | 行号 |
|------|---------|------|
| `bru.setNextRequest` | `packages/bruno-js/src/bru.js` | 380-382 |
| `bru.runner.setNextRequest` | `packages/bruno-js/src/bru.js` | 81-83 |
| 跳转逻辑（执行循环） | `packages/bruno-electron/src/ipc/network/index.js` | 1875-1892 |
| `nextRequestName` 检查 | `packages/bruno-electron/src/ipc/network/index.js` | 1507-1509 |
| `bru.runRequest` 按路径 | `packages/bruno-electron/src/ipc/network/index.js` | 1296-1309 |
| `findItemInCollectionByPathname` | `packages/bruno-electron/src/utils/collection.js` | 606-610 |

### 5.4 改名相关

| 模块 | 文件位置 | 行号 |
|------|---------|------|
| 改显示名 IPC | `packages/bruno-electron/src/ipc/collection.js` | 838-880 |
| 改文件名 IPC | `packages/bruno-electron/src/ipc/collection.js` | 882-979 |
| `requestUids` 缓存 | `packages/bruno-electron/src/cache/requestUids.js` | 1-81 |
