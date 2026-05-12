# 文件系统监听与标签页一致性设计（R3）

聚焦外部重命名、删除后重建场景下的 add/unlink 竞态处理、syncTabUid 触发时机与 pathname 兜底匹配恢复链路。

---

## 1. add/unlink 竞态处理：避免重复节点

### 1.1 问题场景

外部编辑器重命名文件时，文件系统事件触发顺序不确定：
- **理想顺序**：`unlink old-name.bru` → `add new-name.bru`
- **实际可能**：`add new-name.bru` → `unlink old-name.bru`（事件乱序）

如果不处理竞态，会出现：
1. 侧边栏出现重复节点
2. **React key 重复导致渲染崩溃**（"duplicate keys causing react renderer to go mad"）
3. 标签页与文件关联断裂

### 1.2 处理机制源码分析

**文件位置**：`packages/bruno-app/src/providers/ReduxStore/slices/collections/index.js:2719-2760`

```javascript
collectionAddFileEvent: (state, action) => {
  const file = action.payload.file;
  // ... collection root / folder root 处理 ...

  if (collection) {
    const dirname = path.dirname(file.meta.pathname);
    // ... 目录层级处理 ...

    if (file.meta.name != 'folder.bru' && !currentSubItems.find((f) => f.name === file.meta.name)) {
      
      // ===== 竞态检测核心逻辑 =====
      // 先按 UID 查找是否已存在相同节点
      // 注释明确说明：this happens when you rename a file
      // the add event might get triggered first, before the unlink event
      const currentItem = find(currentSubItems, (i) => i.uid === file.data.uid);
      
      if (currentItem) {
        // ===== 存在：做更新（而不是添加）=====
        // 这就是为什么重命名不会产生重复节点的关键
        currentItem.name = file.data.name;
        currentItem.type = file.data.type;
        currentItem.seq = file.data.seq;
        currentItem.tags = file.data.tags;
        currentItem.request = mergeRequestWithPreservedUids(currentItem.request, file.data.request);
        currentItem.filename = file.meta.name;      // ← 文件名更新（pathname 变了）
        currentItem.pathname = file.meta.pathname;  // ← 完整路径更新
        currentItem.settings = file.data.settings;
        currentItem.examples = file.data.examples;
        currentItem.draft = null;
        currentItem.partial = file.partial;
        currentItem.loading = file.loading;
        currentItem.size = file.size;
        currentItem.error = file.error;
        currentItem.isTransient = isTransientFile;
      } else {
        // ===== 不存在：真正添加新节点 =====
        currentSubItems.push({
          uid: file.data.uid,
          name: file.data.name,
          type: file.data.type,
          // ... 完整字段 ...
          pathname: file.meta.pathname,
        });
      }
    }
    addDepth(collection.items);
  }
}
```

### 1.3 竞态时序图

**Case 1: 事件顺序正确（unlink 先到）**
```
T0: unlink event → collectionUnlinkFileEvent
    → findItemInCollectionByPathname(old-pathname)
    → deleteItemInCollectionByPathname
    ✓ 旧节点已删除

T1: add event → collectionAddFileEvent
    → find(currentSubItems, uid) → null
    → push 新节点
    ✓ 无重复
```

**Case 2: 事件乱序（add 先到）**
```
T0: add event → collectionAddFileEvent
    → find(currentSubItems, uid) → 找到旧节点！
    → 执行 IN-PLACE UPDATE（name/pathname/request...）
    → 旧节点被原地修改为新文件名、新路径
    ✓ 没有重复，uid 保持一致

T1: unlink event → collectionUnlinkFileEvent
    → findItemInCollectionByPathname(old-pathname)
    → ❌ 找不到了！因为 T0 已被原地更新为新路径
    → 静默跳过，不执行删除
    ✓ 最终状态正确
```

### 1.4 unlink 事件的查找策略

**collectionUnlinkFileEvent**（`index.js:2863-2874`）：
```javascript
collectionUnlinkFileEvent: (state, action) => {
  const { file } = action.payload;
  const collection = findCollectionByUid(state.collections, file.meta.collectionUid);

  if (collection) {
    // ⚠️ 注意：unlink 只用 pathname 查找！不用 uid！
    const item = findItemInCollectionByPathname(collection, file.meta.pathname);

    if (item) {
      deleteItemInCollectionByPathname(file.meta.pathname, collection);
    }
  }
}
```

**为什么 unlink 只用 pathname 而不用 uid？**
1. 文件已被删除 → 解析不出 uid
2. 竞态场景下（add 先执行）：旧 pathname 已不存在，跳过删除是正确的
3. 正常场景下：pathname 存在 → 正确删除

---

## 2. syncTabUid 触发时机与修正逻辑

### 2.1 问题场景：重命名导致 uid 失配

文件重命名后：
- collection item 的 `uid` 保持不变（来自文件内容解析）
- 但 tab.uid 可能变成了**临时值**或**旧值**
- 标签页显示 "Tab Not Found"

### 2.2 双重查找 + 自动修正机制

**文件位置**：`packages/bruno-app/src/components/RequestTabs/RequestTab/index.js:44-61`

```javascript
// Step 1: 先按 uid 精确匹配
let item = findItemInCollection(collection, tab.uid);

// Step 2: uid 匹配失败时 → 按 pathname 兜底匹配
// ↓ 这就是重命名场景的恢复关键！
if (!item && tab.pathname) {
  item = findItemInCollectionByPathname(collection, tab.pathname);
}

// Step 3: 检测到 uid 失配 → 触发 syncTabUid 修正
useEffect(() => {
  const isRequestType = tab.type === 'request' || /* ... */;

  // 触发条件：
  // 1. 是请求类型 tab
  // 2. tab 有 pathname
  // 3. 找到了对应的 item
  // 4. ⚠️ tab.uid !== item.uid（发生了失配！）
  if (!isRequestType || !tab.pathname || !item?.uid || tab.uid === item.uid) {
    return;
  }

  // 触发同步：把 tab.uid 从旧值更新为 item.uid（来自文件解析的稳定 uid）
  dispatch(syncTabUid({ oldUid: tab.uid, newUid: item.uid }));
  
}, [dispatch, item?.uid, tab.pathname, tab.type, tab.uid]);
```

### 2.3 syncTabUid Reducer 实现

**文件位置**：`packages/bruno-app/src/providers/ReduxStore/slices/tabs.js:432-440`

```javascript
syncTabUid: (state, action) => {
  const { oldUid, newUid } = action.payload;
  const tab = find(state.tabs, (t) => t.uid === oldUid);
  
  if (tab) {
    // 1. 修正 tab 自身的 uid
    tab.uid = newUid;
    
    // 2. ⭐ 同时修正 activeTabUid！
    // 如果当前激活的就是这个 tab，也要一起更新
    // 否则会出现：activeTabUid 是旧值 → 找不到对应 tab → 标签页显示异常
    if (state.activeTabUid === oldUid) {
      state.activeTabUid = newUid;
    }
  }
}
```

### 2.4 修正时序图（重命名场景）

```
T0: 用户在外部重命名文件
    old: GET-users.bru → new: GET-users-v2.bru

T1: add event 先到（竞态）
    → collectionAddFileEvent
    → 按 uid 找到已存在的 item
    → 原地更新：item.name = 'GET-users-v2', item.pathname = 'new-path'
    ✓ collection 侧已更新

T2: React 重渲染 → RequestTab 组件执行
    a) findItemInCollection(collection, tab.uid)
       → 成功！因为 uid 没变，只是 pathname 变了
       → ✅ 跳过 syncTabUid

    ⚠️ 或者（文件被删除后重建，uid 变化）：
    a) findItemInCollection(collection, tab.uid) → null
    b) findItemInCollectionByPathname(collection, tab.pathname) → 找到！
    c) tab.uid !== item.uid → 触发 syncTabUid
    d) tab.uid 被修正，activeTabUid 也同步修正
    ✓ 标签页重新关联成功
```

---

## 3. tab.pathname 兜底匹配恢复链路

### 3.1 完整链路全景

```
                        ┌─────────────────────────┐
                        │   文件系统事件          │
                        │  add / change / unlink  │
                        └────────┬────────────────┘
                                 │
                                 ▼
                        ┌─────────────────────────┐
                        │ collectionAddFileEvent  │
                        │  - uid 存在则 UPDATE    │
                        │  - uid 不存在则 PUSH    │
                        └────────┬────────────────┘
                                 │
                ┌────────────────┴───────────────┐
                │                                │
                ▼                                ▼
    ┌─────────────────────────┐      ┌─────────────────────────┐
    │  tab.uid 精确匹配       │      │  tab.uid 匹配失败       │
    │  findItemInCollection   │      │  (uid 变更 / 临时值)    │
    └────────┬────────────────┘      └────────┬────────────────┘
             │ 成功                            │ 失败
             ▼                                 ▼
    ┌─────────────────────────┐      ┌─────────────────────────┐
    │  正常渲染              │      │  pathname 兜底匹配       │
    │  不触发 syncTabUid     │      │  findItemByPathname     │
    └─────────────────────────┘      └────────┬────────────────┘
                                                │ 找到
                                                ▼
                                      ┌─────────────────────────┐
                                      │  uid 失配检测           │
                                      │  tab.uid !== item.uid   │
                                      └────────┬────────────────┘
                                                │ 需要修正
                                                ▼
                                      ┌─────────────────────────┐
                                      │ dispatch(syncTabUid)    │
                                      │  - 更新 tab.uid         │
                                      │  - 更新 activeTabUid    │
                                      └────────┬────────────────┘
                                                │ 完成
                                                ▼
                                      ┌─────────────────────────┐
                                      │  标签页正常渲染         │
                                      │  关联恢复成功          │
                                      └─────────────────────────┘
```

### 3.2 匹配优先级设计

**Selector 层**（`selectors/tab.js`）也遵循相同的双重匹配逻辑：

```javascript
export const getTabUidForItem = ({ itemUid, itemPathname, collectionUid }) => 
  createSelector([(state) => state.tabs.tabs], (tabs) => {
    // 优先级 1: 按 uid 精确匹配
    const tabByUid = tabs.find((tab) => 
      tab.uid === itemUid && (!collectionUid || tab.collectionUid === collectionUid)
    );
    if (tabByUid) return tabByUid.uid;

    // 优先级 2: 按 pathname 兜底匹配
    if (!itemPathname) return null;
    const tabByPathname = tabs.find((tab) => 
      tab.pathname === itemPathname && (!collectionUid || tab.collectionUid === collectionUid)
    );
    return tabByPathname?.uid || null;
  });
```

### 3.3 恢复失败的降级处理

如果双重匹配都失败了：
1. 渲染 **RequestTabNotFound** 组件
2. 显示 "Request Not Found" 提示
3. 提供关闭按钮让用户手动关闭无效标签页

**文件位置**：`packages/bruno-app/src/components/RequestTabs/RequestTab/RequestTabNotFound.js`

---

## 4. 删除后重建场景的一致性

### 4.1 场景描述

用户操作：
1. 在 Bruno 中打开请求文件（标签页 A 激活）
2. 在外部删除该文件 → 再用相同内容重建（或者 Git 切换分支导致文件先删后加）

### 4.2 一致性保证

```
T0: unlink event 到达
    → collectionUnlinkFileEvent
    → 按 pathname 找到 item
    → deleteItemInCollectionByPathname
    ✗ collection item 被删除了
    ✓ tab 还在！（tabs slice 独立，不会随 collection 删除而关闭）

T1: add event 到达（文件重建）
    → collectionAddFileEvent
    → 按 uid 查找（文件内容相同 → uid 相同！）→ 可能找到也可能找不到
    → 找不到就 push 新节点
    ✓ collection 侧恢复了 item

T2: RequestTab 组件重渲染
    a) findItemInCollection(collection, tab.uid) → 找到！（重建的 item uid 相同）
    b) uid 匹配 → 不触发 syncTabUid
    c) ✅ 标签页自动恢复！用户感觉不到中间被删除过

    ⚠️ 如果重建的文件内容不同（uid 变了）：
    a) uid 匹配失败 → 按 pathname 匹配
    b) 找到新 uid 的 item → 触发 syncTabUid 修正关联
    c) ✅ 仍然可以恢复！
```

### 4.3 关键依赖：UID 稳定性

整个链路的基础假设是：**相同文件内容解析出的 uid 是稳定的**。

UID 生成逻辑（推测）：
- 基于文件内容哈希生成
- 或者是文件元数据中保存的稳定 ID
- 只要内容不变，uid 就不变

这保证了：
1. 删除重建 → uid 相同 → 自动恢复关联
2. 重命名 → uid 相同 → pathname 更新，uid 匹配仍然有效

---

## 5. 边界场景汇总

| 场景 | 处理机制 | 结果 |
|------|---------|------|
| **外部重命名** | add 先到：按 uid 原地更新 pathname + name | ✓ 无重复节点，标签页 uid 匹配正常 |
| **重命名 uid 变化** | uid 匹配失败 → pathname 兜底 → syncTabUid 修正 | ✓ 标签页自动恢复关联 |
| **删除后重建（内容相同）** | uid 相同 → 直接匹配成功 | ✅ 完全无缝，用户无感知 |
| **删除后重建（内容不同）** | uid 匹配失败 → pathname 兜底 → syncTabUid | ✓ 关联恢复，内容更新 |
| **Git 切换分支（批量文件变更）** | 每个文件独立处理竞态 | ✓ 整体一致性保证 |
| **文件移动到其他文件夹** | pathname 变化但 uid 不变 | ✓ uid 匹配仍有效，标签页正常 |
| **真正删除（不重建）** | unlink 删除 item，tab 显示 NotFound | ✓ 提供关闭按钮 |

---

## 6. 架构设计总结

### 三层防御机制

```
┌─────────────────────────────────────────────────────────────────┐
│  L1: 竞态防御（Reducer 层）                                      │
│  collectionAddFileEvent 中按 uid 查找 → 存在则 UPDATE 而非 PUSH  │
│  从根源避免重复节点，不依赖事件顺序                              │
└──────────────────────────────────┬──────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│  L2: 双重匹配（Selector / 组件层）                               │
│  tab.uid 精确匹配 → 失败则 tab.pathname 兜底匹配               │
│  保证重命名、移动、重建等场景下总能找到对应 item                │
└──────────────────────────────────┬──────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│  L3: 自动修正（Effect 层）                                       │
│  检测到 tab.uid !== item.uid → 触发 syncTabUid                  │
│  同时修正 tab.uid 和 activeTabUid，保证激活状态一致性           │
└─────────────────────────────────────────────────────────────────┘
```

### 关键设计决策

| 决策 | 收益 | 代价 |
|------|------|------|
| **add 事件做 upsert 而不是纯 insert** | 解决竞态顺序问题，无重复节点 | 增加 reducer 复杂度 |
| **unlink 只用 pathname 查找，不用 uid** | 文件已删除时 uid 不可用，竞态场景下正确跳过删除 | 极端 edge case 下可能漏删 |
| **tab 保留 uid + pathname 双字段** | 重命名、移动、重建场景下总能兜底关联 | 数据冗余 |
| **syncTabUid 同时修正 activeTabUid** | 避免 "找到了 item 但标签页显示未激活" 不一致 | 需要维护两份数据一致性 |
| **UID 基于内容稳定生成** | 删除重建后自动恢复关联 | 内容变更会导致 uid 变化，需要兜底机制 |

---

**文件位置参考**:
- 竞态更新逻辑: `collections/index.js:2719-2760`
- unlink 删除逻辑: `collections/index.js:2863-2874`
- RequestTab 双重匹配: `RequestTab/index.js:44-61`
- syncTabUid reducer: `tabs.js:432-440`
- tab selector 匹配逻辑: `selectors/tab.js:3-17`
