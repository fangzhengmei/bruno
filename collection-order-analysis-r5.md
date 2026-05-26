# Collection 运行顺序深度分析（R5 修正版）

> **修正说明**：本版本在 R4 基础上补充了三个关键修正：
> 1. 区分了**整集合运行**与**子目录运行**时 `folder` 与 `collection` 的引用关系差异
> 2. 明确了 `requestUids` 缓存的归属（主进程独有，与渲染进程无关）
> 3. 分析了同名请求下 `findIndex` 的 first-match 行为及改名带来的影响

---

## 1. 子目录运行与整集合运行的引用关系差异

### 1.1 调用链入口

**位置**：`packages/bruno-app/src/providers/ReduxStore/slices/collections/actions.js:667-720`

```javascript
export const runCollectionFolder
  = (collectionUid, folderUid, recursive, delay, tags, selectedRequestUids) => (dispatch, getState) => {
    const state = getState();
    const collection = findCollectionByUid(state.collections.collections, collectionUid);
    // ...
    let collectionCopy = cloneDeep(collection);         // 第 1 次隔离：渲染进程内深拷贝
    // ...
    const folder = findItemInCollection(collectionCopy, folderUid);  // folder 是 collectionCopy 内的引用
    // ...
    ipcRenderer.invoke(
      'renderer:run-collection-folder',
      folder,          // 参数1：子目录引用（指向 collectionCopy 内部）
      collectionCopy,  // 参数2：完整 collection 副本
      // ...
    );
};
```

### 1.2 两种运行场景的 IPC 接收端

**位置**：`packages/bruno-electron/src/ipc/network/index.js:1274-1314`

```javascript
ipcMain.handle(
  'renderer:run-collection-folder',
  async (event, folder, collection, environment, runtimeVariables, recursive, delay, tags, selectedRequestUids) => {
    // 经过 IPC 结构化克隆后，folder 和 collection 是独立的对象副本
    // 此时需要判断是哪种场景

    if (!folder) {
      folder = collection;  // 场景 A：整集合运行
    }
    // 场景 B：子目录运行（folder 已有值，是独立副本）
    // ...
  }
);
```

### 1.3 场景 A：整集合运行（folderUid 为 null/undefined）

```
渲染进程：
  folder = findItemInCollection(collectionCopy, null)  → folder = undefined
  IPC 传递：folder = undefined, collection = collectionCopy

主进程（IPC 接收后）：
  folder = undefined
  collection = 结构化克隆后的完整 collection
  if (!folder) { folder = collection; }  ← folder 和 collection 指向同一对象树
    ↓
sortFolder(folder)  → 就地修改 collection.items
getAllRequestsInFolderRecursively(sortedFolder)
  → 收集的 item 是 collection 对象树中的直接引用
    ↓
folderRequests[i] === findItemInCollectionByPathname(collection, path)
  → 同一引用 ✅
```

**关键特性**：`folderRequests` 中的每个元素与 `collection` 对象树中的对应节点是**同一引用**。

### 1.4 场景 B：子目录运行（folderUid 有效）

```
渲染进程：
  folder = findItemInCollection(collectionCopy, folderUid)
    → folder 是 collectionCopy.items 树中的某个节点引用
  IPC 传递：folder = 子目录对象, collection = collectionCopy

主进程（IPC 接收后）：
  结构化克隆：folder 和 collection 被分别序列化为 JSON 再反序列化
    → folder 成为独立对象副本
    → collection 成为独立对象副本
    → folder 不再是 collection 内部的引用
    ↓
folder = 独立的子目录对象（包含完整的子树）
collection = 独立的完整 collection 对象
    ↓
sortFolder(folder)  → 就地修改 folder 对象（不影响 collection）
getAllRequestsInFolderRecursively(sortedFolder)
  → 收集的 item 是 folder 独立对象树中的引用
    ↓
folderRequests[i] !== findItemInCollectionByPathname(collection, path)
  → 不同引用 ❌（但属性值相同）
```

**关键特性**：`folderRequests` 中的元素与 `collection` 对象树中的节点是**不同对象**，但 `pathname`、`name`、`uid` 等属性值相同。

### 1.5 两种场景对比表

| 维度 | 整集合运行 | 子目录运行 |
|------|-----------|-----------|
| `folder` 来源 | `folder = collection` | 独立的子目录副本 |
| `sortFolder` 影响 | 修改 `collection.items` | 仅修改 `folder`（不影响 `collection`） |
| `folderRequests[i]` 与 `collection` | **同一引用** | **不同对象**（属性值相同） |
| `runRequestByItemPathname` 查找 | 找到与 `folderRequests[i]` 同一对象 | 找到不同对象（属性相同） |
| 排序对 `collection` 的副作用 | ✅ 有副作用 | ❌ 无副作用 |

### 1.6 对执行的实际影响

虽然引用不同，但对执行无实际影响，因为：
1. `runRequestByItemPathname` 使用 `cloneDeep(findItemInCollectionByPathname(collection, itemPathname))`，克隆后传入执行
2. 两个对象的属性值（`request`、`name`、`uid` 等）完全相同
3. 执行过程中不依赖对象引用一致性

---

## 2. requestUids 缓存归属纠正

### 2.1 定义位置

**位置**：`packages/bruno-electron/src/cache/requestUids.js:13-14`

```javascript
const requestUids = new Map();    // pathname → uid
const exampleUids = new Map();    // `${pathname}-${index}` → uid
```

**关键事实**：此文件位于 `packages/bruno-electron/src/cache/`，属于**主进程（Electron main process）**。

### 2.2 使用方分析

| 使用文件 | 所属进程 | 使用的函数 |
|---------|---------|-----------|
| `packages/bruno-electron/src/utils/collection.js` | 主进程 | `getRequestUid`, `getExampleUid` |
| `packages/bruno-electron/src/ipc/collection.js` | 主进程 | `moveRequestUid`, `deleteRequestUid`, `syncExampleUidsCache` |
| `packages/bruno-electron/src/app/collection-watcher.js` | 主进程 | `getRequestUid` |

**结论**：`requestUids` 缓存**仅在主进程中使用**，渲染进程完全不感知此缓存。

### 2.3 缓存的作用

```
文件系统 → 主进程解析 → requestUids.get(pathname) → 返回稳定的 uid
                                ↑
                           如果不存在，生成新 uid 并缓存

改名/移动 → moveRequestUid(oldPath, newPath) → 保持 uid 不变
                                ↓
                           确保草稿状态不丢失
```

### 2.4 与渲染进程的关系

渲染进程通过 Redux store 管理 collection 数据，UID 的来源：

1. **主进程解析文件时**：调用 `getRequestUid(pathname)` 生成/获取 UID
2. **IPC 传递到渲染进程**：解析后的 collection 数据（含 UID）通过 IPC 发送
3. **渲染进程 Redux store**：存储收到的 collection 数据
4. **改名操作**：渲染进程发 IPC 请求 → 主进程更新文件 → 主进程更新 `requestUids` 缓存 → 重新解析 → IPC 回传新数据 → Redux 更新

### 2.5 运行时队列与 requestUids 的关系

运行时队列（`folderRequests`）使用的 UID 来自：
- 整集合运行：来自 `collection` 对象树（IPC 克隆后的副本）
- 子目录运行：来自 `folder` 独立副本

这些 UID 是在 collection 加载/解析时由 `requestUids` 缓存生成的**快照值**。运行过程中 `requestUids` 缓存的更新不会影响已构建的队列。

---

## 3. 同名请求下 findIndex 的 first-match 行为

### 3.1 跳转查找逻辑

**位置**：`packages/bruno-electron/src/ipc/network/index.js:1883`

```javascript
const nextRequestIdx = folderRequests.findIndex(
  (request) => request.name === nextRequestName
);
```

**关键特性**：
- 使用 `===` 严格相等比较
- `findIndex` 返回**第一个**匹配元素的索引
- 如果没有匹配，返回 `-1`

### 3.2 递归模式下的 first-match 顺序

递归模式下，`folderRequests` 由 `getAllRequestsInFolderRecursively` 深度优先遍历构建：

```
Collection
├── Folder A (seq: 1)
│   ├── Request A1 (seq: 1, name: "login")   ← index 0
│   └── Request A2 (seq: 2, name: "getUser")  ← index 1
├── Folder B (seq: 2)
│   └── Request B1 (seq: 1, name: "login")    ← index 2
└── Request C (seq: 3, name: "listUsers")     ← index 3
```

如果有两个请求同名 `"login"`：
- `findIndex` 找到 index 0（Folder A 中的 Request A1）
- 无论 `bru.setNextRequest("login")` 在哪里被调用，始终跳转到 Request A1
- Request B1 永远不会被 `setNextRequest("login")` 选中

### 3.3 非递归模式下的 first-match 顺序

非递归模式下，`folderRequests` 由 `sortByNameThenSequence` 排序：

```
Folder X（运行目标）
├── Request X1 (seq: 1, name: "login")   ← index 0
├── Request X2 (seq: 2, name: "getUser")  ← index 1
├── Request X3 (no seq, name: "login")   ← index 2（无 seq，字母排序兜底）
```

如果有两个请求同名 `"login"`：
- Request X1（有 seq）优先级高，排在前面
- `findIndex` 找到 index 0
- Request X3 永远不会被 `setNextRequest("login")` 选中

### 3.4 标签过滤和手动选择对 first-match 的影响

标签过滤和手动选择过滤会**缩减 `folderRequests` 数组**，可能改变 first-match 的结果：

```
原始 folderRequests: [Req A1("login"), Req A2("getUser"), Req B1("login")]

标签过滤后（排除 B1）: [Req A1("login"), Req A2("getUser")]
  → setNextRequest("login") 仍匹配 index 0

手动选择后（仅选中 B1）: [Req B1("login")]
  → setNextRequest("login") 现在匹配 index 0（B1）
  → 手动选择可以改变 first-match 的结果
```

### 3.5 改名对 first-match 的影响

#### 场景：运行中将 Request A1 改名为 "authenticate"

```
运行前：
  folderRequests = [
    A1(name="login"),    ← setNextRequest("login") 匹配此
    A2(name="getUser"),
    B1(name="login")
  ]

运行中改名：A1: "login" → "authenticate"
  但 folderRequests 是快照，不更新
  folderRequests 仍然 = [
    A1(name="login"),    ← setNextRequest("login") 仍匹配此
    A2(name="getUser"),
    B1(name="login")
  ]

实际行为：
  setNextRequest("login")      → 匹配 A1（快照旧名称）
  setNextRequest("authenticate") → 不匹配（快照中没有此名称）
  setNextRequest("authenticate") → currentRequestIndex++（顺序继续）
```

#### 场景：运行中将 Request B1 改名为 "authenticate"

```
运行前：
  folderRequests = [
    A1(name="login"),    ← index 0
    A2(name="getUser"),  ← index 1
    B1(name="login")     ← index 2
  ]

运行中改名：B1: "login" → "authenticate"
  folderRequests 快照不变

实际行为：
  setNextRequest("login")           → 匹配 A1（index 0）
  setNextRequest("login") 再次调用  → 匹配 A1（index 0），可能导致死循环！
  setNextRequest("authenticate")    → 不匹配（B1 在快照中仍叫 "login"）
```

### 3.6 死循环风险与防护

**位置**：`packages/bruno-electron/src/ipc/network/index.js:1876-1879`

```javascript
nJumps++;
if (nJumps > 10000) {
  throw new Error('Too many jumps, possible infinite loop');
}
```

如果请求 A 的脚本调用 `bru.setNextRequest("A")`（跳转到自身），会导致：
- 每次执行 A 后又跳回 A
- 形成死循环
- 防护：`nJumps` 计数器超过 10000 次后抛出异常

### 3.7 同名请求的实际行为总结表

| 操作 | 递归模式 | 非递归模式 |
|------|---------|-----------|
| `setNextRequest("重复名称")` | 匹配**深度优先顺序中第一个**出现的请求 | 匹配**排序后第一个**出现的请求 |
| 运行中改名后调用旧名称 | ✅ 匹配快照旧名称 | ✅ 匹配快照旧名称 |
| 运行中改名后调用新名称 | ❌ 不匹配（顺序继续） | ❌ 不匹配（顺序继续） |
| 运行中将第一个匹配改名 | `setNextRequest("旧名称")` 仍匹配第一个（快照不变） | 同左 |
| 运行中将第二个匹配改名 | 无影响（第一个仍在快照中） | 无影响 |

---

## 4. 完整运行器调用链与数据流图

```
[UI 层] 用户点击 "Run"
    │
    ▼
[渲染进程] actions.js:runCollectionFolder
    │
    ├─ findCollectionByUid(Redux, collectionUid)
    │
    ├─ cloneDeep(collection) → collectionCopy        （第 1 次隔离）
    │
    ├─ findItemInCollection(collectionCopy, folderUid) → folder
    │    ├─ 整集合：folder = undefined
    │    └─ 子目录：folder = collectionCopy 内的引用
    │
    └─ IPC.invoke('renderer:run-collection-folder', folder, collectionCopy, ...)
         │
         ▼ （第 2 次隔离：IPC 结构化克隆）
    │
    ▼
[主进程] ipcMain.handle('renderer:run-collection-folder')
    │
    ├─ 接收独立副本：folder, collection
    │
    ├─ if (!folder) folder = collection
    │    ├─ 整集合：folder === collection（同一引用）
    │    └─ 子目录：folder ≠ collection（不同对象）
    │
    ├─ sortFolder(folder)
    │    ├─ 整集合：就地修改 collection.items
    │    └─ 子目录：仅修改 folder 独立对象
    │
    ├─ getAllRequestsInFolderRecursively(sortedFolder)
    │    └─ 收集请求到 folderRequests（引用指向 folder 树）
    │
    ├─ 标签过滤 filter()  ← 不改变顺序
    │
    ├─ 手动选择过滤+排序 filter().sort()  ← 完全覆盖顺序
    │
    ├─ folderRequests 构建完成（不可变快照）
    │
    ├─ runRequestByItemPathname 闭包（使用 collection 参数）
    │    └─ findItemInCollectionByPathname(collection, pathname)
    │        ├─ 整集合：找到与 folderRequests[i] 同一对象
    │        └─ 子目录：找到不同对象（属性相同）
    │
    └─ while(currentRequestIndex < folderRequests.length)
         │
         ├─ 执行 folderRequests[currentRequestIndex]
         │
         ├─ 检查 nextRequestName
         │    ├─ undefined → currentRequestIndex++
         │    ├─ null → break
         │    └─ name → findIndex(name) 找第一个匹配
         │        ├─ 找到 → 跳转
         │        └─ 未找到 → currentRequestIndex++
         │
         └─ 检查 stopExecution → break
    │
    ▼
[外部操作] 运行中改名/移动/删除
    │
    ├─ 文件系统：✅ 更新
    ├─ requestUids 缓存（主进程）：✅ 更新
    ├─ 渲染进程 Redux：✅ 更新（文件监视器）
    └─ 主进程运行闭包（folder, collection, folderRequests）：❌ 完全隔离
```

---

## 5. R4 → R5 修正总结

| 修正点 | R4 版本 | R5 版本（修正后） |
|--------|--------|-----------------|
| folder 与 collection 引用 | 笼统描述为"同一引用" | 区分两种场景：整集合运行=同一引用，子目录运行=不同对象 |
| requestUids 缓存归属 | 未明确进程归属 | 明确为主进程独有，渲染进程不感知 |
| 同名请求跳转 | 未分析 | 明确 `findIndex` first-match 行为及改名对快照的影响 |
| 排序副作用 | 未明确 | 整集合运行时 `sortFolder` 会修改 `collection.items`，子目录运行则不会 |

---

## 6. 代码溯源索引

### 6.1 调用链与引用关系

| 模块 | 文件位置 | 行号 |
|------|---------|------|
| 渲染进程 runCollectionFolder | `packages/bruno-app/src/providers/ReduxStore/slices/collections/actions.js` | 667-720 |
| 渲染进程 cloneDeep | `packages/bruno-app/src/providers/ReduxStore/slices/collections/actions.js` | 678 |
| 渲染进程 findItemInCollection | `packages/bruno-app/src/providers/ReduxStore/slices/collections/actions.js` | 687 |
| IPC 传递参数 | `packages/bruno-app/src/providers/ReduxStore/slices/collections/actions.js` | 702-713 |
| 主进程 IPC handler | `packages/bruno-electron/src/ipc/network/index.js` | 1274 |
| folder = collection（整集合） | `packages/bruno-electron/src/ipc/network/index.js` | 1312-1314 |

### 6.2 requestUids 缓存

| 模块 | 文件位置 | 行号 |
|------|---------|------|
| 缓存定义 | `packages/bruno-electron/src/cache/requestUids.js` | 13-14 |
| `getRequestUid` | `packages/bruno-electron/src/cache/requestUids.js` | 17-26 |
| `moveRequestUid` | `packages/bruno-electron/src/cache/requestUids.js` | 28-35 |
| `deleteRequestUid` | `packages/bruno-electron/src/cache/requestUids.js` | 37-39 |
| 主进程使用 | `packages/bruno-electron/src/utils/collection.js` | 3（import） |
| 主进程使用 | `packages/bruno-electron/src/ipc/collection.js` | 62（import） |
| 主进程使用 | `packages/bruno-electron/src/app/collection-watcher.js` | 21（import） |

### 6.3 跳转与 first-match

| 模块 | 文件位置 | 行号 |
|------|---------|------|
| `findIndex` 跳转 | `packages/bruno-electron/src/ipc/network/index.js` | 1883 |
| `nJumps` 死循环防护 | `packages/bruno-electron/src/ipc/network/index.js` | 1876-1879 |
| `bru.setNextRequest` | `packages/bruno-js/src/bru.js` | 380-382 |
| `bru.runner.setNextRequest` | `packages/bruno-js/src/bru.js` | 81-83 |

### 6.4 排序与收集

| 模块 | 文件位置 | 行号 |
|------|---------|------|
| `sortFolder` | `packages/bruno-electron/src/utils/collection.js` | 712-727 |
| `getAllRequestsInFolderRecursively` | `packages/bruno-electron/src/utils/collection.js` | 729-747 |
| `sortByNameThenSequence` | `packages/bruno-electron/src/utils/collection.js` | 855-894 |
| 递归/非递归分支 | `packages/bruno-electron/src/ipc/network/index.js` | 1327-1340 |
