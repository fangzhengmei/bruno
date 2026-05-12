# 外部重命名场景的恢复机制深度分析（R5）

纠正了 pathname 兜底的错误假设，明确 tab.pathname 永不更新的设计缺陷、syncTabUid 只改 uid 的副作用、以及多次重命名场景下的兜底失效路径。

---

## 1. 核心设计缺陷：tab.pathname 永不更新

### 1.1 关键发现

**collection item 的 pathname 会更新**（`collections/index.js:2731, 2852`）：
```javascript
// collectionAddFileEvent 中（重命名竞态场景）
currentItem.pathname = file.meta.pathname;  // ← 用新路径更新

// collectionChangeFileEvent 中
item.pathname = file.meta.pathname;
```

**但 tab 的 pathname 永不更新**！

检查 `tabs.js` 所有 reducer：
- `addTab`: 创建时设置 pathname 一次
- `closeTabs`: 关闭标签
- `reorderTabs`: 重新排序
- `syncTabUid`: ⚠️ 只改 uid 和 activeTabUid，**不改 pathname**！
- `restoreTabs`: 恢复快照
- 等等...

**结论**：tab.pathname 是"一次性"值，创建后永远不变！

---

## 2. syncTabUid 只改 uid 的后果

### 2.1 syncTabUid 实现回顾

**文件位置**：`packages/bruno-app/src/providers/ReduxStore/slices/tabs.js:432-440`

```javascript
syncTabUid: (state, action) => {
  const { oldUid, newUid } = action.payload;
  const tab = find(state.tabs, (t) => t.uid === oldUid);
  if (tab) {
    tab.uid = newUid;              // ← 只更新 uid
    if (state.activeTabUid === oldUid) {
      state.activeTabUid = newUid; // ← 只更新 activeTabUid
    }
    // ⚠️ 完全没有更新 tab.pathname！
  }
},
```

### 2.2 后果分析

| 字段 | syncTabUid 后是否更新 | 后果 |
|-----|----------------------|------|
| `tab.uid` | ✅ 更新为 item.uid | 下次渲染时 uid 直接匹配 |
| `state.activeTabUid` | ✅ 更新 | 激活状态保持 |
| `tab.pathname` | ❌ 不变，仍是旧路径 | 「埋下地雷」，下次重命名时兜底失效 |

---

## 3. 单次重命名 vs 多次重命名

### 3.1 场景 1：单次外部重命名（可恢复）

```
初始状态:
  tab.uid = old-uid-123
  tab.pathname = /path/GET-users.bru  ← 创建时设置
  item.uid = old-uid-123
  item.pathname = /path/GET-users.bru

T0: 外部编辑器重命名
    old: /path/GET-users.bru
    new: /path/GET-users-v2.bru

T1: unlink 事件 → 删除旧 item

T2: add 事件 → 创建新 item（新 uid）
    item.uid = new-uid-456
    item.pathname = /path/GET-users-v2.bru  ← 路径更新了

T3: RequestTab 渲染
    a) findItemInCollection(collection, tab.uid=old-uid-123)
       → null ❌ 找不到

    b) findItemInCollectionByPathname(collection, tab.pathname=/path/GET-users.bru)
       → null ❌ 也找不到！

    ⚠️ 等等！这里有问题！

    c) collection-watcher 的 add 事件是按 uid 做 upsert，不是 add！
       实际上，由于竞态时序，可能是原地更新了同一个 item！
```

### 3.2 重新审视竞态时序

让我们重新分析重命名的**真实竞态路径**（先 add 后 unlink）：

```
初始状态:
  tab.uid = uid-123
  tab.pathname = /path/old.bru
  item.uid = uid-123
  item.pathname = /path/old.bru

T0: 外部重命名 old.bru → new.bru

T1: add 事件先到达（竞态）
    → collectionAddFileEvent
    → 按 uid 查找 item: uid-123 存在！
    → 原地更新 item:
        item.uid = getRequestUid(newPathname) = uid-789  ← 新 uid！
        item.pathname = newPathname = /path/new.bru
    ⚠️ 一个 item 从 uid-123 变成了 uid-789！

T2: RequestTab 渲染
    a) findItemInCollection(collection, tab.uid=uid-123)
       → null ❌ uid 变了

    b) findItemInCollectionByPathname(collection, tab.pathname=/path/old.bru)
       → null ❌ item 的 pathname 也变了！

    c) ⚠️ 兜底失效！直接显示 RequestNotFound！
```

### 3.3 场景 2：什么时候 pathname 兜底能命中？

**只有一种情况**：add 事件创建了**新的 item**，而不是原地更新。

```
初始状态:
  tab.uid = old-uid-123
  tab.pathname = /path/test.bru

T0: unlink 事件先到达（竞态正确顺序）
    → collectionUnlinkFileEvent
    → 按 pathname 找到并删除 item
    → collection 中没有这个 item 了

T1: add 事件后到达
    → collectionAddFileEvent
    → 按 uid 查找: old-uid-123 不存在（已删）
    → push 新 item
        new-item.uid = getRequestUid(newPathname) = new-uid-456
        new-item.pathname = /path/test.bru  ← 路径没变！
        （只改文件名不改路径的重命名）

T2: RequestTab 渲染
    a) findItemInCollection(collection, tab.uid=old-uid-123)
       → null ❌ 找不到

    b) findItemInCollectionByPathname(collection, tab.pathname=/path/test.bru)
       → 找到 new-item！✅
       因为只改了文件名，父路径没变！

    c) tab.uid !== item.uid → 触发 syncTabUid
    d) tab.uid 更新为 new-uid-456
    e) ✅ 关联恢复成功！
```

**关键点**：
- pathname 兜底命中的前提是：**文件只改了文件名，父目录路径不变**
- 且 unlink 先执行，add 后执行（不是原地 upsert）

---

## 4. 重命名场景的完整恢复矩阵

### 4.1 按操作类型分类

| 重命名场景 | pathname 是否变化 | 竞态顺序 | 兜底是否命中 | 最终状态 |
|-----------|------------------|---------|------------|---------|
| **只改文件名**（`a.bru`→`b.bru`） | 父路径不变，文件名变 | unlink 先 | ✅ pathname 兜底命中 → syncTabUid | 可恢复 |
| **只改文件名**（`a.bru`→`b.bru`） | 父路径不变，文件名变 | add 先（upsert） | ❌ uid 变了，pathname 也变了 | RequestNotFound |
| **改目录 + 文件名**（`a/x.bru`→`b/y.bru`） | 完全变 | 任何顺序 | ❌ 完全找不到 | RequestNotFound |
| **只改目录**（`a/x.bru`→`b/x.bru`） | 父路径变，文件名不变 | 任何顺序 | ❌ pathname 完全不同 | RequestNotFound |

### 4.2 多次重命名的连锁失效

**第二次重命名必然失效**：

```
第 1 次重命名（假设运气好，兜底命中了）:
  tab.pathname = /path/old1.bru  ← 创建时设置，永不更新
  item.pathname = /path/old2.bru ← 第一次重命名后的新路径
  兜底命中 → syncTabUid 只改了 uid，没改 pathname
  tab.pathname 还是 /path/old1.bru！⚠️

第 2 次重命名:
  item.pathname = /path/old3.bru ← 第二次重命名后的路径
  tab.pathname = /path/old1.bru  ← 还是最初的路径！
  a) uid 匹配失败（uid 又变了）
  b) pathname 兜底: /path/old1.bru vs /path/old3.bru
     → ❌ 完全不匹配！
  → 🔴 RequestNotFound！
```

**结论**：即使第一次重命名运气好恢复了，第二次重命名 100% 会兜底失效！

---

## 5. 哪些场景仍可恢复，哪些会落到 RequestNotFound

### 5.1 ✅ 仍可恢复的场景

| 场景 | 恢复机制 | 备注 |
|-----|---------|------|
| **内部重命名** | moveRequestUid 保证 uid 不变 | 完美恢复 |
| **外部删除→立即重建**（路径不变） | uid 缓存未清理，uid 不变 | 完美恢复 |
| **外部只改文件名** + **unlink 先到** | pathname 兜底命中（父路径不变） + syncTabUid | 可恢复一次，第二次重命名失效 |
| **Git 切换分支，文件内容变路径不变** | uid 不变或 pathname 兜底 | 大概率可恢复 |

### 5.2 ❌ 必然落到 RequestNotFound 的场景

| 场景 | 原因 |
|-----|------|
| **外部移动文件到其他文件夹** | pathname 完全改变，兜底失效 |
| **外部重命名 + 竞态 add 先到**（upsert 模式） | uid 和 pathname 同时变，双重不匹配 |
| **同一个文件第二次外部重命名** | tab.pathname 仍是最初值，与当前 pathname 完全脱节 |
| **文件被移动 + 重命名**（双重变化） | pathname 完全不同，兜底失效 |

---

## 6. 架构设计的根本问题

### 6.1 问题根源

```
tabs slice 设计缺陷:
  ├─ tab.pathname 只在创建时设置，永不更新
  ├─ syncTabUid 只修正 uid，不修正 pathname
  └─ 没有专门的 syncTabPathname reducer

后果:
  tab.pathname 是「过时的快照」，只能用一次
  只要文件路径发生变化，最终必然兜底失效
```

### 6.2 为什么不更新 tab.pathname？

可能的设计考虑：
1. pathname 本来就是用作兜底的「最后一道防线」，不是主关联键
2. uid 才是主关联键，内部操作保证 uid 不变，pathname 兜底只是为外部操作擦屁股
3. 外部文件操作本来就是低频场景，体验降级可接受
4. 修复需要在多个地方更新 pathname，容易引入新 bug

---

## 7. 关键事实对照表（最终版）

| 陈述 | 真伪 | 说明 |
|-----|------|------|
| tab.pathname 会随文件重命名自动更新 | ❌ 假 | 创建后永不更新 |
| syncTabUid 会同时更新 tab.pathname | ❌ 假 | 只更新 uid 和 activeTabUid |
| 外部重命名总能通过 pathname 兜底恢复 | ❌ 假 | 取决于竞态顺序和路径变化程度 |
| 同一个文件第二次外部重命名必然失效 | ✅ 真 | tab.pathname 仍是最初值，完全脱节 |
| 外部移动文件到其他文件夹必然失效 | ✅ 真 | pathname 完全改变，兜底失效 |
| 只改文件名不改目录，unlink 先到可恢复 | ✅ 真 | 父路径不变，兜底可命中 |
| 只改文件名不改目录，add 先到（upsert）也失效 | ✅ 真 | uid 和 pathname 同时变，双重不匹配 |

---

## 8. 可能的改进方向

### 方案 A：syncTabUid 同时更新 pathname

```javascript
syncTabUid: (state, action) => {
  const { oldUid, newUid, newPathname } = action.payload;  // + 传新路径
  const tab = find(state.tabs, (t) => t.uid === oldUid);
  if (tab) {
    tab.uid = newUid;
    if (newPathname) {
      tab.pathname = newPathname;  // + 同时更新 pathname
    }
    if (state.activeTabUid === oldUid) {
      state.activeTabUid = newUid;
    }
  }
},
```

### 方案 B：collection-watcher 检测 rename 事件

在主进程中追踪文件移动，当检测到可能的 rename 时：
- 调用 moveRequestUid 保持 uid 不变
- 发送专门的 collectionRenameFileEvent 到渲染进程
- 渲染进程同时更新 item.pathname 和 tab.pathname

### 方案 C：兜底匹配时同时更新 tab.pathname

在 RequestTab 组件中，pathname 兜底命中后：
- 不仅调用 syncTabUid 修正 uid
- 同时调用新的 syncTabPathname 修正 pathname
- 保证下一次重命名时兜底仍然有效

---

**文件位置参考**:
- collection item pathname 更新: `collections/index.js:2731, 2852`
- syncTabUid 只改 uid: `tabs.js:432-440`
- tabs slice 无 pathname 更新逻辑: 搜索 `tab\.pathname.*=` 无结果
- RequestTab 双重匹配: `RequestTab/index.js:44-61`
