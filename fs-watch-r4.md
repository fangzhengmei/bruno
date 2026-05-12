# 文件系统监听与 requestUids 缓存机制（R4）

纠正了 UID 生成逻辑的错误假设，明确 requestUids 缓存的 pathname-key 设计、内部/外部操作的差异、以及删除后重建场景的标签页恢复链路。

---

## 1. requestUids 缓存机制核心实现

### 1.1 缓存设计目标

**文件位置**：`packages/bruno-electron/src/cache/requestUids.js:1-11`

```javascript
/**
 * we maintain a cache of request uids to ensure that we
 * preserve the same uid for a request even when the request
 * moves to a different location
 *
 * In the past, we used to generate unique ids based on the
 * pathname of the request, but we faced problems when implementing
 * functionality where the user can move the request to a different
 * location. In that case, the uid would change, and we would
 * lose the request's draft state if the user has made some changes
 */
```

**设计目标**：
- 保证**请求移动到不同位置**时 UID 保持不变
- 避免草稿状态丢失
- 保证标签页关联不中断

### 1.2 三大核心 API

```javascript
const requestUids = new Map();  // key: pathname, value: uuid

// API 1: 首次访问时分配，后续返回缓存值
const getRequestUid = (pathname) => {
  let uid = requestUids.get(pathname);
  if (!uid) {
    uid = uuid();              // ⚠️ 随机生成，与内容无关！
    requestUids.set(pathname, uid);
  }
  return uid;
};

// API 2: 重命名/移动时迁移（保持 uid 不变）
const moveRequestUid = (oldPathname, newPathname) => {
  const uid = requestUids.get(oldPathname);
  if (uid) {
    requestUids.delete(oldPathname);
    requestUids.set(newPathname, uid);  // 同一个 uid 迁移到新 key
  }
};

// API 3: 删除时清理缓存
const deleteRequestUid = (pathname) => {
  requestUids.delete(pathname);
};
```

**关键纠正**：
- ❌ **之前错误**："UID 基于文件内容稳定生成"
- ✅ **真实实现**：`uuid()` 随机生成，与文件内容无关
- ✅ **缓存 key**：`pathname`（文件路径）
- ✅ **一致性保证**：同一路径返回相同 uid；路径变更需显式调用 moveRequestUid

---

## 2. 内部操作 vs 外部操作的差异

### 2.1 内部操作（Bruno UI 内重命名/移动）

**触发路径**：用户在侧边栏右键重命名/移动 → IPC `rename-item` handler

**文件位置**：`packages/bruno-electron/src/ipc/collection.js:948-961`

```javascript
// 重命名文件的关键步骤（内部操作）
const data = await fs.promises.readFile(oldPath, 'utf8');
const jsonData = parseRequest(data, { format });
jsonData.name = newName;

// ⭐ 关键：文件系统操作前先迁移 uid！
moveRequestUid(oldPath, newPath);  // ← 显式调用迁移

// 然后才执行文件系统操作
const content = stringifyRequest(jsonData, { format });
await fs.promises.unlink(oldPath);
await writeFile(newPath, content);
```

**结果**：
- UID 保持不变
- tab.uid 与 item.uid 始终匹配
- ✅ 标签页正常，draft 状态完全保留

### 2.2 外部操作（外部编辑器重命名/移动）

**触发路径**：chokidar 监听到 unlink + add 事件 → collection-watcher 处理

**关键事实**：`collection-watcher.js` 中**没有**调用 `moveRequestUid`！

```javascript
// collection-watcher.js 中只有 getRequestUid，没有 move/delete
const { getRequestUid } = require('../cache/requestUids');

// 每一次 add 事件都会调用 getRequestUid
file.data.uid = getRequestUid(pathname);  // ← 总是用当前 pathname 查缓存
```

**外部重命名时序**：
```
T0: 外部编辑器重命名文件
    old: /path/GET-users.bru
    new: /path/GET-users-v2.bru

T1: unlink 事件到达
    → collectionUnlinkFileEvent
    → 按 oldPathname 删除 item
    ⚠️ 没有调用 deleteRequestUid！缓存还在！

T2: add 事件到达
    → collectionAddFileEvent
    → getRequestUid(newPathname)
    → ❌ newPathname 不在缓存中 → 生成新的 uid！
    → 用新 uid 创建 item
```

**结果**：
- UID 改变了！
- tab.uid ≠ item.uid
- 但 tab.pathname 兜底匹配 + syncTabUid 可以恢复关联

---

## 3. 删除后重建场景详细分析

### 3.1 内部删除 vs 外部删除

| 删除方式 | 调用 deleteRequestUid? | 缓存状态 | 重建时 uid |
|---------|-----------------------|---------|-----------|
| **内部删除**（UI 内删除） | ✅ 是（`ipc/collection.js:1019`） | oldPathname 缓存被清理 | 生成新 uid |
| **外部删除**（文件系统删除） | ❌ 否 | oldPathname 缓存仍然存在 | 重建时 uid 不变！ |

### 3.2 场景 1：外部删除后立即重建（内容相同）

```
T0: 用户在外部删除文件 /path/test.bru
    → unlink 事件到达 collection-watcher
    → collectionUnlinkFileEvent 删除 item
    ⚠️ 没有调用 deleteRequestUid！缓存仍保留：/path/test.bru → old-uid-123

T1: 用户立即重建文件（内容相同或不同）
    → add 事件到达 collection-watcher
    → getRequestUid('/path/test.bru')
    → ✅ 命中缓存！返回 old-uid-123
    → item.uid = old-uid-123
    → tab.uid = old-uid-123
    → ✅ uid 完全匹配！
    → ✅ 标签页无缝恢复，draft 保留（如果内容相同）
```

### 3.3 场景 2：外部删除后应用重启再重建

```
T0: 外部删除文件，应用未重启
    → unlink 事件，缓存保留

T1: 退出应用 → 内存清空 → requestUids Map 完全丢失

T2: 重启应用，重新加载 collection
    → 对每个文件调用 getRequestUid(pathname)
    → 缓存为空，生成新 uid new-uid-456

T3: 删除重建文件
    → unlink + add
    → getRequestUid 命中新缓存
    → uid 不变（但与重启前不同）
```

### 3.4 场景 3：内部删除后重建

```
T0: 在 UI 内删除文件
    → deleteRequestUid(pathname) 清理缓存
    → item 从 collection 删除

T1: Git 恢复文件（或外部重建）
    → add 事件到达
    → getRequestUid(pathname)
    → ❌ 缓存已清理，生成新 uid new-uid-789
    → tab.uid = old-uid
    → item.uid = new-uid-789
    → ⚠️ uid 不匹配！
    → 但 tab.pathname 兜底匹配 + syncTabUid 修正
    → ✅ 仍然可以恢复关联！
```

---

## 4. 标签页恢复链路完整分析

### 4.1 路径匹配的兜底作用

**文件位置**：`packages/bruno-app/src/components/RequestTabs/RequestTab/index.js:44-61`

```javascript
// Step 1: 先按 uid 匹配
let item = findItemInCollection(collection, tab.uid);

// Step 2: uid 匹配失败时，用 pathname 兜底
// ⭐ 这就是 uid 变更后仍然能恢复的关键！
if (!item && tab.pathname) {
  item = findItemInCollectionByPathname(collection, tab.pathname);
}

// Step 3: 检测失配并修正
useEffect(() => {
  if (/* ... */ !item?.uid || tab.uid === item.uid) {
    return;
  }
  // uid 不同，触发同步修正
  dispatch(syncTabUid({ oldUid: tab.uid, newUid: item.uid }));
}, [/* ... */]);
```

### 4.2 各种场景下的恢复效果

| 操作场景 | uid 是否变化 | 恢复机制 | 最终效果 |
|---------|-------------|---------|---------|
| **内部重命名/移动** | ❌ 不变（moveRequestUid） | uid 直接匹配 | ✅ 完美，无感知 |
| **外部重命名** | ✅ 变化（无 moveRequestUid） | pathname 兜底 + syncTabUid | ✅ 可恢复，uid 修正 |
| **外部删除→立即重建** | ❌ 不变（缓存未清理） | uid 直接匹配 | ✅ 完美，无感知 |
| **外部删除→重启→重建** | ✅ 变化（缓存丢失） | pathname 兜底 + syncTabUid | ✅ 可恢复，uid 修正 |
| **内部删除→外部重建** | ✅ 变化（缓存已清理） | pathname 兜底 + syncTabUid | ✅ 可恢复，uid 修正 |

### 4.3 pathname 兜底的边界情况

**兜底失效的唯一情况**：文件重命名 + 移动到其他文件夹（pathname 完全改变）

```
原文件: /path/a/GET-test.bru
新文件: /path/b/POST-test.bru
```

此时：
- `tab.pathname` 是旧路径 `/path/a/GET-test.bru`
- `findItemInCollectionByPathname` 找不到匹配项
- ❌ 兜底失效，标签页显示 NotFound
- ✅ 但用户可以手动关闭，不影响其他功能

---

## 5. 架构设计总结

### 5.1 requestUids 设计的权衡

| 设计决策 | 收益 | 代价 |
|---------|------|------|
| **pathname 作为 cache key** | 简单高效，同路径一致性保证 | 外部重命名/移动 uid 会变 |
| **随机 uuid 生成，与内容无关** | 实现简单，无哈希冲突问题 | 删除重建（缓存清理后）uid 变 |
| **只有内部操作调用 move/delete** | 内部体验完美 | 外部操作一致性依赖兜底机制 |
| **unlink 事件不清理缓存** | 外部删除后立即重建 uid 不变，体验好 | 内存泄漏风险（但实际可接受） |

### 5.2 四层一致性保障

```
┌─────────────────────────────────────────────────────────────────┐
│  L1: requestUids 缓存（主进程）                                  │
│  getRequestUid: 同 pathname 同 uid                              │
│  moveRequestUid: 内部重命名 uid 不变                            │
└──────────────────────────────────┬──────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│  L2: add 事件 upsert（渲染进程 Reducer）                        │
│  按 uid 查找，存在则原地更新，避免重复节点                      │
└──────────────────────────────────┬──────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│  L3: 双重匹配（组件层）                                          │
│  uid 精确匹配 → 失败则 pathname 兜底匹配                        │
└──────────────────────────────────┬──────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│  L4: syncTabUid 自动修正（Effect 层）                           │
│  检测 uid 失配，同时修正 tab.uid 和 activeTabUid               │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 为什么不监听 rename 事件？

chokidar 实际上支持 `rename` 事件，但 Bruno 没有使用，原因可能是：
1. `rename` 事件在不同平台/文件系统下行为不一致
2. unlink + add 的组合处理更可靠、更通用
3. pathname 兜底 + syncTabUid 已经足够处理 uid 变更场景
4. 外部操作本来就不多，体验降级可接受

---

## 6. 关键事实对照表

| 陈述 | 真伪 | 说明 |
|-----|------|------|
| UID 基于文件内容哈希生成 | ❌ 假 | 随机 uuid() 生成，与内容无关 |
| requestUids cache key 是 pathname | ✅ 真 | Map<pathname, uid> |
| 内部重命名 uid 不变 | ✅ 真 | 显式调用 moveRequestUid |
| 外部重命名 uid 不变 | ❌ 假 | 没有 moveRequestUid，uid 变 |
| 外部删除后立即重建 uid 不变 | ✅ 真 | 缓存未清理，add 时命中 |
| 内部删除后重建 uid 不变 | ❌ 假 | deleteRequestUid 清理了缓存 |
| unlink 事件会清理 requestUids 缓存 | ❌ 假 | 只有内部删除才清理 |
| uid 变了标签页就一定找不到 | ❌ 假 | pathname 兜底 + syncTabUid |

---

**文件位置参考**:
- requestUids 缓存实现: `packages/bruno-electron/src/cache/requestUids.js`
- 内部重命名调用 moveRequestUid: `packages/bruno-electron/src/ipc/collection.js:956`
- 内部删除调用 deleteRequestUid: `packages/bruno-electron/src/ipc/collection.js:1010,1019`
- collection-watcher 只调用 getRequestUid: `packages/bruno-electron/src/app/collection-watcher.js:113,154,187,416`
- 标签页双重匹配逻辑: `packages/bruno-app/src/components/RequestTabs/RequestTab/index.js:44-61`
