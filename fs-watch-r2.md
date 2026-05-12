# 文件系统监听与内存模型增量更新链路设计（R2）

基于代码的精确分析，修正了字段级更新路径与标签页同步机制的实现细节。

---

## 1. 监听器实现（回顾）

### 1.1 事件监听与去抖

使用 `chokidar` 监听文件变更，内置 `awaitWriteFinish` 等待文件写入稳定：
- `stabilityThreshold: 80ms` - 80ms 内文件无变化才触发事件
- `pollInterval: 10ms` - 每 10ms 轮询文件状态

额外去抖层：
- `unlink` 事件延迟 100ms 处理，避免文件重建时状态不一致

### 1.2 事件分发流程

```
文件变更 (add/change/unlink)
    ↓
collection-watcher.js 解析文件内容
    ↓
构造 payload: { file: { meta: {...}, data: {...} } }
    ↓
IPC 发送: main:collection-tree-updated (type, file)
    ↓
useIpcEvents.js 接收并 dispatch Redux action
    ↓
collectionChangeFileEvent / collectionUnlinkFileEvent
```

---

## 2. 字段级增量更新实现

### 2.1 精确查找节点

**文件位置**: `packages/bruno-app/src/providers/ReduxStore/slices/collections/index.js:2829`

```javascript
collectionChangeFileEvent: (state, action) => {
  const { file } = action.payload;
  const collection = findCollectionByUid(state.collections, file.meta.collectionUid);
  
  // ⚠️ 关键: 通过 UID 查找，不是通过 pathname
  // 保证即使文件移动/重命名，也能正确关联
  const item = findItemInCollection(collection, file.data.uid);
  
  if (item) {
    // 增量更新逻辑...
  }
}
```

### 2.2 两阶段更新策略

#### Phase 1: 仅排序变化（seq-only update）

当 `areItemsTheSameExceptSeqUpdate(item, file.data)` 返回 true 时：
- 说明除了 `seq` 字段外，文件内容与内存数据完全一致
- 这种情况通常发生在拖拽排序后重新保存文件

**处理逻辑** (`index.js:2835-2842`):
```javascript
if (areItemsTheSameExceptSeqUpdate(item, file.data)) {
  // 只更新 seq 字段
  item.seq = file.data.seq;
  
  // 如果有草稿，也更新草稿的 seq
  if (item?.draft) {
    item.draft.seq = file.data.seq;
  }
  
  // 如果草稿与文件内容完全匹配（只seq不同），清空草稿
  // 这意味着外部保存的就是草稿中的内容
  if (item?.draft && areItemsTheSameExceptSeqUpdate(item?.draft, file.data)) {
    item.draft = null;
  }
}
```

#### Phase 2: 完整字段更新

当文件确实有内容变化时，逐个字段更新（不替换整个对象）：

**更新字段列表** (`index.js:2844-2858`):
```javascript
} else {
  // 基础元数据字段 - 直接赋值替换
  item.name = file.data.name;
  item.type = file.data.type;
  item.seq = file.data.seq;
  item.tags = file.data.tags;
  item.settings = file.data.settings;
  item.examples = file.data.examples;
  
  // 文件路径元数据
  item.filename = file.meta.name;
  item.pathname = file.meta.pathname;
  
  // ⭐ request 字段：UID 保留合并
  // 保持 params/headers 等数组内元素的 UID 不变
  // 避免 React 组件卸载重挂载，保留光标位置、滚动位置
  item.request = mergeRequestWithPreservedUids(item.request, file.data.request);
  
  // 📝 草稿保留逻辑
  // 只有当草稿与磁盘文件完全匹配时才清空草稿
  // 否则保留草稿（用户正在编辑的内容不被外部覆盖）
  if (item.draft && areItemsTheSameExceptSeqUpdate(item.draft, file.data)) {
    item.draft = null;
  }
}
```

### 2.3 `areItemsTheSameExceptSeqUpdate` 深度比较

**文件位置**: `packages/bruno-app/src/utils/collections/index.js:1041-1062`

这是核心的比较函数，决定了：
1. 是否只发生了排序变化
2. 草稿是否应该被清空

```javascript
export const areItemsTheSameExceptSeqUpdate = (_item1, _item2) => {
  let item1 = cloneDeep(_item1);
  let item2 = cloneDeep(_item2);

  // 1. 移除 seq 字段（排序变化不视为内容变化）
  delete item1.seq;
  delete item2.seq;

  // 2. 移除 draft 字段（草稿不参与比较）
  delete item1.draft;
  delete item2.draft;

  // 3. 转换为文件系统存储格式（去除 UI 临时状态）
  // 例如：response、运行时状态等不会保存到磁盘
  item1 = transformRequestToSaveToFilesystem(item1);
  item2 = transformRequestToSaveToFilesystem(item2);

  // 4. 移除所有 UID
  // UID 是内存中分配的，文件中不存储
  deleteUidsInItem(item1);
  deleteUidsInItem(item2);

  // 5. 深度比较
  return isEqual(item1, item2);
};
```

**关键点**：
- 比较的是「文件存储格式」，不是完整内存对象
- UID 不参与比较（文件中不存储）
- `seq` 不参与比较（排序独立于内容）
- `draft` 不参与比较（草稿本身是临时状态）

### 2.4 UID 保留合并机制

**文件位置**: `packages/bruno-app/src/providers/ReduxStore/slices/collections/index.js:74-113`

```javascript
const preserveUidsAtPaths = (existing, updated, paths) => {
  if (!existing || !updated) return updated;
  const merged = cloneDeep(updated);

  paths.forEach((path) => {
    const newArray = get(merged, path);
    const existingArray = get(existing, path, []);

    if (Array.isArray(newArray) && newArray.length) {
      // 按位置保留 UID
      // 假设：文件重新解析后，数组顺序不变
      // 效果：React keys 稳定，不触发卸载重渲染
      set(merged, path, newArray.map((item, i) => 
        existingArray[i]?.uid 
          ? { ...item, uid: existingArray[i].uid } 
          : item
      ));
    }
  });

  return merged;
};

// Request 层级需要保留 UID 的路径
const REQUEST_UID_PATHS = [
  'params',               // 请求参数
  'headers',              // 请求头
  'vars.req',             // 前置变量
  'vars.res',             // 响应变量
  'assertions',           // 断言
  'body.formUrlEncoded',  // form-urlencoded
  'body.multipartForm',   // multipart/form-data
  'body.file'             // 文件上传
];
```

---

## 3. 标签页同步与状态保持

### 3.1 标签页数据结构

**文件位置**: `packages/bruno-app/src/providers/ReduxStore/slices/tabs.js`

```javascript
// Tab 不存储实际请求数据，只存储关联引用和 UI 状态
{
  uid: 'request-uid-123',           // 关联 item.uid
  collectionUid: 'collection-uid',  // 关联 collection.uid
  type: 'http-request',             // 请求类型
  pathname: '/path/to/file.bru',    // 文件路径（冗余关联，双重保险）
  
  // ========== UI 状态 ==========
  // 这些状态完全独立于文件内容
  // 文件变化时不会被覆盖
  requestPaneWidth: 600,            // 请求面板宽度
  requestPaneHeight: null,          // 请求面板高度
  requestPaneCollapsed: false,      // 请求面板是否折叠
  responsePaneCollapsed: false,     // 响应面板是否折叠
  requestPaneTab: 'params',         // 当前激活的请求子 tab
  responsePaneTab: 'response',      // 当前激活的响应子 tab
  responseFormat: null,             // 响应显示格式
  responseViewTab: null,            // 响应视图 tab
  responseFilter: '',               // 响应过滤文本
  responseFilterExpanded: false,    // 过滤器是否展开
  tableColumnWidths: {},            // 各表格列宽记忆
  scriptPaneTab: null,              // Script 面板 tab
  preview: true,                    // 是否为预览模式（单击打开）
  gqlDocsOpen: false,               // GraphQL 文档是否展开
  docsEditing: false,               // 文档是否正在编辑
}
```

### 3.2 标签页与 Collection 的关联机制

**Selector 层**: `packages/bruno-app/src/selectors/tab.js`

```javascript
// 查找匹配的 tab：先按 UID，再按 pathname
// 支持文件移动/重命名后仍然能正确关联
export const getTabUidForItem = ({ itemUid, itemPathname, collectionUid }) => 
  createSelector([(state) => state.tabs.tabs], (tabs) => {
    // 优先按 UID 匹配
    const tabByUid = tabs.find((tab) => 
      tab.uid === itemUid && (!collectionUid || tab.collectionUid === collectionUid)
    );
    if (tabByUid) return tabByUid.uid;

    // UID 不匹配时，按 pathname 兜底匹配
    if (!itemPathname) return null;
    const tabByPathname = tabs.find((tab) => 
      tab.pathname === itemPathname && (!collectionUid || tab.collectionUid === collectionUid)
    );
    return tabByPathname?.uid || null;
  });
```

### 3.3 完整同步链路

```
  ┌─────────────────────────────────────────────────────────────┐
  │                     标签页 (tabs slice)                     │
  │  ┌───────────────────────────────────────────────────────┐│
  │  │ 存储: UID 关联 + UI 状态（面板宽、当前 tab 等）       ││
  │  │ 完全独立于文件内容，文件变更时不会被修改               ││
  │  └───────────────────────────────────────────────────────┘│
  └───────────────────────────┬─────────────────────────────────┘
                              │
                              │ 通过 UID/pathname 关联
                              │
  ┌───────────────────────────▼─────────────────────────────────┐
  │                  Collection Items 存储                      │
  │  ┌───────────────────────────────────────────────────────┐│
  │  │ item = {                                             ││
  │  │   uid, name, type, seq, tags,                       ││
  │  │   request, settings, examples,                      ││
  │  │   draft, 👈 用户正在编辑的内容（文件变化时条件保留）││
  │  │   response, 👈 响应结果（文件变化时不丢失）         ││
  │  │   ... 其他运行时状态                                ││
  │  │ }                                                    ││
  │  └───────────────────────────────────────────────────────┘│
  └───────────────────────────┬─────────────────────────────────┘
                              │
                              │ ⬆️ 增量更新
                              │
  ┌───────────────────────────▼─────────────────────────────────┐
  │               collectionChangeFileEvent                    │
  │  ┌───────────────────────────────────────────────────────┐│
  │  │ 1. 按 UID 查找 item                                   ││
  │  │ 2. areItemsTheSameExceptSeqUpdate?                   ││
  │  │    ├─ YES: 只更新 seq，保留 draft                    ││
  │  │    └─ NO: 逐个字段更新 name/type/seq/tags/request...││
  │  │ 3. 条件清空 draft（只有当 draft == 文件内容时）      ││
  │  │ 4. ✅ response、面板状态等完全不受影响               ││
  │  └───────────────────────────────────────────────────────┘│
  └───────────────────────────┬─────────────────────────────────┘
                              │
                              │ React Redux 重渲染
                              │
  ┌───────────────────────────▼─────────────────────────────────┐
  │                    编辑器组件层级                           │
  │                                                             │
  │  RequestTab (Container)                                     │
  │     ├─ 根据 activeTabUid 找到 tab 对象                      │
  │     ├─ 根据 tab.uid/pathname 从 collection 找到 item        │
  │     └─ 将 item 传给子组件                                   │
  │                                                             │
  │  RequestPane (子组件)                                       │
  │     ├─ 渲染 item.request / item.draft.request              │
  │     └─ 从 tab 对象读取 UI 状态（宽度、当前 tab 等）         │
  │                                                             │
  │  ResponsePane (子组件)                                      │
  │     └─ 渲染 item.response（文件变化时保留）                 │
  │                                                             │
  │  ✅ 效果：文件变化只更新数据，React diff 更新 DOM，          │
  │           不卸载组件，光标位置、滚动位置完全保留            │
  └─────────────────────────────────────────────────────────────┘
```

### 3.4 状态保留清单

| 状态类型 | 存储位置 | 文件变化时是否保留 | 备注 |
|---------|---------|------------------|------|
| **响应结果** | `item.response` | ✅ 完全保留 | 不在更新字段列表中，不受影响 |
| **草稿内容** | `item.draft` | ✅ 条件保留 | 只有当 draft 与文件内容完全匹配时才清空 |
| **面板宽度/高度** | `tab.requestPaneWidth` | ✅ 完全保留 | tabs slice 独立存储 |
| **当前激活的 Tab** | `tab.requestPaneTab` | ✅ 完全保留 | tabs slice 独立存储 |
| **面板折叠状态** | `tab.requestPaneCollapsed` | ✅ 完全保留 | tabs slice 独立存储 |
| **表格列宽** | `tab.tableColumnWidths` | ✅ 完全保留 | tabs slice 独立存储 |
| **光标位置** | DOM 状态 | ✅ 间接保留 | UID 不变 → React 不卸载组件 |
| **滚动位置** | DOM 状态 | ✅ 间接保留 | UID 不变 → React 不卸载组件 |
| **输入焦点** | DOM 状态 | ✅ 间接保留 | UID 不变 → React 不卸载组件 |

---

## 4. 草稿保留条件详解

### 4.1 保留草稿的场景

当外部编辑文件时，草稿在以下情况下会被**保留**：

1. **用户正在编辑**：
   - 用户在 Bruno 中修改了请求（创建了 draft）
   - 外部编辑器修改了文件的其他部分
   - `areItemsTheSameExceptSeqUpdate(draft, file.data) === false`
   - ✅ 草稿保留，用户的编辑不丢失

2. **只是排序变化**：
   - 拖拽调整了请求顺序，seq 变化
   - `areItemsTheSameExceptSeqUpdate(item, file.data) === true`
   - ✅ 草稿保留，只更新 draft.seq

### 4.2 清空草稿的场景

草稿只在一种情况下被清空：

- `item.draft` 存在，**且**
- `areItemsTheSameExceptSeqUpdate(item.draft, file.data) === true`

这意味着：**磁盘文件的内容与草稿完全一致**。

典型场景：
- 用户在 Bruno 中编辑后保存（草稿写入磁盘）
- 或者外部编辑器保存的内容恰好与草稿相同

此时清空草稿是安全的，因为草稿内容已经持久化。

---

## 5. 边缘情况处理

### 5.1 文件重命名 / 移动

- Tab 通过 `uid` 匹配，pathname 只是兜底
- 文件移动/重命名后，只要 uid 不变，tab 仍然正确关联
- `item.pathname` 会更新为新路径，但 tab.uid 匹配仍然有效

### 5.2 文件在外部被删除后重建

- `unlink` 事件延迟 100ms 处理
- 如果文件立即重建，第二个 `add` 事件会在 unlink 处理前到达
- UID 相同情况下会被当作更新而非删除重建

### 5.3 大文件异步解析

- 大文件使用 Worker 线程解析
- 先发送部分元数据到 UI
- 完整解析完成后再发送完整数据
- UI 先显示占位，后续平滑更新

---

## 6. 架构总结

### 三层分离设计

```
┌─────────────────────────────────────────────────────┐
│  UI 状态层 (tabs slice)                             │
│  ─────────────────────────────────────────────────  │
│  面板宽度、当前 Tab、折叠状态、列宽记忆             │
│  ✅ 完全独立，文件变化零影响                         │
└─────────────────────────────────────────────────────┘
                          ↓ 关联（uid/pathname）
┌─────────────────────────────────────────────────────┐
│  业务数据层 (collections slice)                    │
│  ─────────────────────────────────────────────────  │
│  request, settings, examples, tags                 │
│  ✅ 字段级增量更新，UID 保留合并                    │
└─────────────────────────────────────────────────────┘
                          ↓ 条件保留
┌─────────────────────────────────────────────────────┐
│  临时状态层 (draft / response)                      │
│  ─────────────────────────────────────────────────  │
│  用户未保存编辑、请求响应结果                       │
│  ✅ 只在内容完全匹配时才清空 draft                  │
│  ✅ response 完全不受文件变化影响                   │
└─────────────────────────────────────────────────────┘
```

### 关键设计决策

| 决策 | 收益 | 代价 |
|------|------|------|
| 按位置保留 UID | DOM 不卸载，编辑状态 100% 保留 | 假设文件解析后数组顺序不变 |
| 字段级更新而非整体替换 | 响应结果等状态不丢失 | 需要明确维护更新字段列表 |
| 草稿条件清空 | 用户正在输入时不会被意外打断 | 外部编辑与草稿冲突时需要手工合并 |
| Tab 只存引用不存数据 | Tab 状态完全独立 | 需要 selector 层做关联查询 |

---

**文件位置参考**:
- 增量更新逻辑: `packages/bruno-app/src/providers/ReduxStore/slices/collections/index.js:2801-2862`
- 比较函数: `packages/bruno-app/src/utils/collections/index.js:1041-1062`
- Tab 数据结构: `packages/bruno-app/src/providers/ReduxStore/slices/tabs.js`
- Tab Selector: `packages/bruno-app/src/selectors/tab.js`
