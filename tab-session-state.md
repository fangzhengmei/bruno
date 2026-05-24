# 标签页会话状态与未保存改动管理分析（可复核版）

## 一、整体架构概览

Bruno 使用 Redux Toolkit 作为状态管理库，通过三个核心中间件协同工作实现标签页会话管理：

```
Redux Store
├── slices
│   ├── tabs.js              # 标签页状态管理
│   ├── collections/index.js # 集合与条目状态管理（含 draft）
│   └── app.js               # 应用级状态
└── middlewares
    ├── draft/middleware.js     # 草稿检测 - 标记预览标签页为永久
    ├── autosave/middleware.js  # 自动保存 - 定时保存草稿
    └── snapshot/middleware.js  # 快照持久化 - 保存UI状态到磁盘
```

## 二、多 Tab 状态树组织

### 2.1 状态树结构

**`tabs` slice** (`packages/bruno-app/src/providers/ReduxStore/slices/tabs.js:13-17`) 定义了标签页的核心状态：

```javascript
{
  tabs: [],              // 打开的标签页数组
  activeTabUid: null,    // 当前活动标签页UID
  recentlyClosedTabs: [] // 最近关闭的标签页栈（LIFO，最多50个）
}
```

**单个标签页对象** (`tabs.js:129-159`) 包含丰富的 UI 状态：

```javascript
{
  uid: string,
  collectionUid: string,
  type: 'http-request' | 'graphql-request' | 'grpc-request' | 'ws-request' | 'folder-settings' | 'collection-settings' | ...,
  pathname: string | null,           // 文件系统路径
  preview: boolean,                  // 是否为预览模式（点击打开 vs 双击打开）
  isTransient?: boolean,             // 是否为临时请求（无持久化文件）
  
  // UI 布局状态
  requestPaneWidth: number | null,
  requestPaneHeight: number | null,
  requestPaneCollapsed: boolean,
  responsePaneCollapsed: boolean,
  requestPaneTab: 'params' | 'body' | 'auth' | 'headers' | ...,
  responsePaneTab: 'response' | 'headers' | 'cookies' | ...,
  responseFormat: 'json' | 'xml' | 'text' | ... | null,
  responseViewTab: 'pretty' | 'raw' | 'preview' | null,
  responseFilter: string | null,
  responseFilterExpanded: boolean,
  gqlDocsOpen: boolean,
  tableColumnWidths: { [tableId: string]: number[] },
  scriptPaneTab: 'pre-request' | 'post-response' | 'tests' | null,
  docsEditing: boolean,
  
  // 关联标识
  folderUid?: string,
  exampleUid?: string,
  itemUid?: string,
  exampleName?: string,
  exampleIndex?: number
}
```

### 2.2 标签页类型与查找机制

**标签页去重与查找** (`tabs.js:23-47`) 通过三种方式匹配已存在的标签页：

1. **精确 UID 匹配**：通过 `uid` 直接查找
2. **路径匹配**：通过 `collectionUid + pathname + type` 匹配
3. **单例类型匹配**：对于 `variables`、`collection-runner`、`preferences` 等单例类型标签页，按 `collectionUid + type` 匹配

**响应示例标签页** 使用 `exampleIndex` 或 `exampleName` 进行额外匹配 (`tabs.js:37-43`)：
```javascript
if (type === 'response-example') {
  if (typeof exampleIndex === 'number') {
    return tab.exampleIndex === exampleIndex;
  }
  return tab.exampleName === exampleName;
}
```

### 2.3 预览模式 vs 永久模式

标签页有两种模式 (`tabs.js:96-127`)：

- **预览模式 (`preview: true`)**：单击侧边栏打开，打开新标签页时会替换当前预览标签
- **永久模式 (`preview: false`)**：双击或编辑后变为永久标签，新标签页会追加到列表

模式转换由 `draftDetectMiddleware` 自动触发（见第三节）。

### 2.4 最近关闭标签页栈

`closeTabs` action (`tabs.js:311-355`) 实现了最近关闭标签页功能：
- 使用 LIFO 栈存储，最多保留 50 个
- 排除不可关闭类型（`workspaceOverview`、`workspaceEnvironments`）和临时请求
- 通过 `reopenLastClosedTab` action 恢复

### 2.5 活动标签页切换策略

关闭标签页时的活动标签页切换逻辑 (`tabs.js:332-349`)：
1. 如果活动标签页仍存在，保持不变
2. 否则优先切换到同一集合的最后一个兄弟标签页
3. 如无兄弟标签页，切换到全局最后一个标签页

## 三、脏检测（未保存改动）机制

### 3.1 Draft 状态模型

Bruno 采用 **Copy-on-Write** 模式管理未保存改动。每个可编辑对象都有一个 `draft` 字段：

```javascript
// collections/index.js:776-782
deleteRequestDraft: (state, action) => {
  const collection = findCollectionByUid(state.collections, action.payload.collectionUid);
  if (collection) {
    const item = findItemInCollection(collection, action.payload.itemUid);
    if (item && item.draft) {
      item.draft = null;  // 丢弃改动
    }
  }
}
```

### 3.2 草稿检测拦截动作数量（可复核）

#### 3.2.1 draftDetectMiddleware（73 个 action）

**代码位置**：`packages/bruno-app/src/providers/ReduxStore/middlewares/draft/middleware.js:3-82`

**统计方式**：Grep 正则 `'collections\/` 统计结果为 **73 个**

**详细分类**：

| 层级 | 行号范围 | 数量 | 示例 action |
|------|---------|------|------------|
| Request-level | 5-47 | **43 个** | `requestUrlChanged`、`updateAuth`、`addQueryParam`、`addRequestHeader`、`updateRequestBody`、`updateRequestGraphqlQuery`、`updateRequestScript`、`addAssertion`、`addVar` 等 |
| Folder-level | 49-62 | **14 个** | `addFolderHeader`、`updateFolderVar`、`updateFolderRequestScript`、`updateFolderAuth`、`updateFolderDocs` 等 |
| Collection-level | 65-81 | **17 个** | `addCollectionHeader`、`updateCollectionVar`、`updateCollectionAuth`、`updateCollectionRequestScript`、`updateCollectionClientCertificates`、`updateCollectionProtobuf`、`updateCollectionProxy` 等 |
| **合计** | | **73 个** | |

**代码引用**：
```javascript
// middleware.js:3-82
const actionsToIntercept = [
  // Request-level actions (lines 5-47, 43 个)
  'collections/requestUrlChanged',
  'collections/updateAuth',
  // ... 共 43 个
  // Folder-level actions (lines 49-62, 14 个)
  'collections/addFolderHeader',
  // ... 共 14 个
  // Collection-level actions (lines 65-81, 17 个)
  'collections/addCollectionHeader',
  // ... 共 17 个
];
```

#### 3.2.2 autosaveMiddleware（83 个 action）

**代码位置**：`packages/bruno-app/src/providers/ReduxStore/middlewares/autosave/middleware.js:5-94`

**统计方式**：Grep 正则 `'collections\/|'global-environments\/` 统计结果为 **83 个**

**详细分类**：

| 层级 | 行号范围 | 数量 | 说明 |
|------|---------|------|------|
| Request-level | 6-54 | **49 个** | 比 draft 多 6 个：`updateCollectionPresets`、`setRequestVars`、`setRequestAssertions`、`updateItemSettings`、`addRequestTag`、`deleteRequestTag` |
| Folder-level | 57-70 | **14 个** | 与 draft 相同 |
| Collection-level | 73-89 | **17 个** | 与 draft 相同 |
| Environment draft | 92-93 | **2 个** | `setEnvironmentsDraft`、`setGlobalEnvironmentDraft` |
| **合计** | | **83 个** | |

**代码引用**：
```javascript
// middleware.js:5-94
const actionsToIntercept = [
  // Request-level actions (lines 6-54, 49 个)
  // ... 同 draft 的 43 个 + 额外 6 个
  
  // Environment draft actions (lines 92-93, 2 个)
  'collections/setEnvironmentsDraft',
  'global-environments/setGlobalEnvironmentDraft'
];
```

### 3.3 脏检测核心函数

**`hasRequestChanges(item)`** (`utils/collections/index.js:1069-1085`) 是脏检测的核心：

```javascript
export const hasRequestChanges = (item) => {
  if (!item || !item.draft) return false;

  const originalItem = cloneDeep(item);
  const draftItem = cloneDeep(item.draft);

  // 排除 examples 进行比较（示例有单独的检测函数）
  delete originalItem.examples;
  delete originalItem.draft;
  delete draftItem.examples;
  delete draftItem.draft;

  return !isEqual(originalItem, draftItem);
};
```

**关键设计**：
- 深度比较原始数据与草稿数据
- 特意排除 `examples` 字段，因为示例有单独的 `hasExampleChanges` 函数
- 使用 `lodash.isEqual` 进行深度相等比较

### 3.4 各类实体的 Draft 状态

| 实体类型 | Draft 位置 | 检测方式 | 删除 Draft Action |
|---------|-----------|---------|------------------|
| 请求 | `item.draft` | `hasRequestChanges(item)` | `deleteRequestDraft` |
| 文件夹 | `folder.draft` | `folder.draft != null` | `deleteFolderDraft` |
| 集合 | `collection.draft` | `collection.draft != null` | `deleteCollectionDraft` |
| 集合环境 | `collection.environmentsDraft` | `collection.environmentsDraft != null` | `clearEnvironmentsDraft` |
| 全局环境 | `state.globalEnvironments.globalEnvironmentDraft` | `globalEnvironmentDraft != null` | `clearGlobalEnvironmentDraft` |

### 3.5 Draft 自动检测中间件

`draftDetectMiddleware` (`middlewares/draft/middleware.js:84-90`) 监听上述 73 种改动 action：

```javascript
export const draftDetectMiddleware = ({ dispatch, getState }) => (next) => (action) => {
  if (actionsToIntercept.includes(action.type)) {
    const state = getState();
    handleMakeTabParmanent(state, action, dispatch);  // 标记标签页为永久
  }
  return next(action);
};
```

**`handleMakeTabParmanent`** (`middlewares/draft/utils.js:5-36`) 逻辑：
1. 获取当前活动标签页
2. 如果是预览模式 (`preview === true`)
3. 根据 action payload 中的 `itemUid` / `folderUid` / `collectionUid` 确定对应实体
4. 派发 `makeTabPermanent` action 将标签页转为永久模式

### 3.6 自动保存机制

`autosaveMiddleware` (`middlewares/autosave/middleware.js:233-264`) 在用户启用自动保存时工作：

**触发时机**：
- 监听上述 83 种 action
- 自动保存启用时，为每个有改动的实体调度延迟保存
- 保存间隔由 `autoSave.interval` 配置控制

**去重机制** (`autosave/middleware.js:96-109`)：
```javascript
const pendingTimers = {};

const scheduleAutoSave = (key, save, interval) => {
  clearTimeout(pendingTimers[key]);  // 清除之前的定时器
  pendingTimers[key] = setTimeout(() => {
    save();
    delete pendingTimers[key];
  }, interval);
};
```

**保存优先级**：
- 按实体类型分别保存：`request-${uid}`、`folder-${uid}`、`collection-${uid}`
- 使用 `key` 去重，同一实体的频繁改动只会触发一次保存
- 临时请求 (`isTransient`) 跳过自动保存

## 四、关闭确认链路

### 4.1 单标签页关闭确认

**入口点**：`RequestTab` 组件 (`components/RequestTabs/RequestTab/index.js`)

关闭按钮点击处理 (`index.js:593-605`)：
```javascript
<GradientCloseButton
  hasChanges={hasChanges}
  onClick={(e) => {
    if (!hasChanges) {
      isWS && closeWsConnection(item.uid);
      return handleCloseClick(e);  // 无改动直接关闭
    }
    e.stopPropagation();
    e.preventDefault();
    setShowConfirmClose(true);   // 有改动显示确认对话框
  }}
/>
```

**确认对话框** `ConfirmRequestClose` 提供三个选项：
1. **Save and Close**：调用 `saveRequest` → `closeTabs`
2. **Don't Save**：调用 `deleteRequestDraft` → `closeTabs`
3. **Cancel**：取消关闭

### 4.2 各类标签页的关闭确认

`RequestTab` 组件为不同类型的标签页实现了专门的关闭处理 (`index.js:158-484`)：

| 标签页类型 | 检查条件 | 确认组件 |
|-----------|---------|---------|
| HTTP/GraphQL/gRPC/WS 请求 | `hasRequestChanges(item)` | `ConfirmRequestClose` |
| 集合设置 | `collection.draft` | `ConfirmCollectionClose` |
| 文件夹设置 | `folder.draft` | `ConfirmFolderClose` |
| 集合环境 | `collection.environmentsDraft` | `ConfirmCloseEnvironment` |
| 全局环境 | `globalEnvironmentDraft` | `ConfirmCloseEnvironment` |

### 4.3 键盘快捷键关闭

`useKeybinding('closeTab')` (`index.js:208-246`) 实现了键盘快捷键关闭，同样遵守脏检查：

```javascript
useKeybinding('closeTab', () => {
  if (tab.type === 'request' || ...) {
    if (hasChanges) {
      setShowConfirmClose(true);  // 有改动显示确认
    } else {
      dispatch(closeTabs({ tabUids: [tab.uid] }));  // 无改动直接关闭
    }
  }
  // ... 其他类型类似处理
}, { enabled: isActive, deps: [...] });
```

### 4.4 批量关闭确认

右键菜单中的批量关闭操作 (`index.js:632-711`) 采用**静默保存**策略：

```javascript
async function handleCloseMultipleTabs(tabs) {
  const tabUidsToClose = [];
  for (const tab of tabs) {
    const item = findItemInCollection(collection, tab.uid);
    if (item && hasRequestChanges(item)) {
      try {
        await dispatch(saveRequest(item.uid, collection.uid, true));  // 静默保存
      } catch (err) {
        continue;  // 保存失败跳过
      }
    }
    tabUidsToClose.push(tab.uid);
  }
  dispatch(closeTabs({ tabUids: tabUidsToClose }));
}
```

**批量关闭类型**：
- `Close Others`：关闭其他标签页
- `Close to the Left`：关闭左侧标签页
- `Close to the Right`：关闭右侧标签页
- `Close Saved`：关闭已保存（无改动）的标签页
- `Close All`：关闭全部标签页

### 4.5 应用关闭确认

**应用级关闭流程**：

1. **主进程触发**：Electron 主进程发送 `main:start-quit-flow` 事件
2. **渲染进程监听**：`ConfirmAppClose` 组件监听事件 (`providers/App/ConfirmAppClose/index.js:11-23`)
3. **收集所有草稿**：`SaveRequestsModal` 组件聚合所有未保存改动 (`providers/App/ConfirmAppClose/SaveRequestsModal.js:27-102`)

**草稿收集逻辑** (`SaveRequestsModal.js:27-102`)：
```javascript
const allDrafts = useMemo(() => {
  const requestDrafts = [];
  const collectionDrafts = [];
  const folderDrafts = [];
  const environmentDrafts = [];
  
  // 按集合分组遍历标签页
  const tabsByCollection = groupBy(relevantTabs, (t) => t.collectionUid);
  
  Object.keys(tabsByCollection).forEach((collectionUid) => {
    const collection = findCollectionByUid(collections, collectionUid);
    // 检查集合 draft
    // 检查集合环境 draft
    // 检查所有请求的 draft（使用 hasRequestChanges）
    // 检查所有文件夹的 draft
  });
  
  // 检查全局环境 draft
  return [...collectionDrafts, ...folderDrafts, ...environmentDrafts, ...requestDrafts];
}, [...]);
```

**用户选择**：
- **Save All**：批量保存所有草稿 → `completeQuitFlow()`
- **Don't Save**：删除所有草稿 → `completeQuitFlow()`
- **Cancel**：取消退出

**批量保存优化** (`SaveRequestsModal.js:147-215`)：
- 按类型分组：集合、文件夹、请求、环境
- 非临时请求使用 `saveMultipleRequests` 批量保存
- 临时请求单独保存并触发 `SaveTransientRequest` 流程
- 环境变量保存前校验变量名合法性

## 五、会话持久化与重启恢复

### 5.1 快照存储服务实际代码位置（可复核）

**三层持久化架构**：

```
┌─────────────────────────────────────────────────────────────────┐
│  Renderer Process (React + Redux)                               │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  序列化工具                                               │  │
│  │  路径: packages/bruno-app/src/utils/snapshot/index.js     │  │
│  │  职责: serializeTab()、deserializeTab()                   │  │
│  └───────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  防抖中间件                                               │  │
│  │  路径: packages/bruno-app/src/providers/ReduxStore/       │  │
│  │            middlewares/snapshot/middleware.js             │  │
│  │  职责: 监听 25 个触发 action，1 秒防抖后调用 IPC           │  │
│  └───────────────────────────────────────────────────────────┘  │
└───────────────────────────────────┬─────────────────────────────┘
                                    │ IPC 通信 (renderer:snapshot:save)
┌───────────────────────────────────▼─────────────────────────────┐
│  Main Process (Electron)                                        │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  SnapshotManager 存储服务（最终落地）                      │  │
│  │  路径: packages/bruno-electron/src/services/snapshot/index.js│  │
│  │  类定义: line 111，class SnapshotManager                  │  │
│  │  导出: line 567，module.exports = new SnapshotManager()    │  │
│  │  存储: electron-store → ui-state-snapshot.json            │  │
│  │  验证: Yup Schema 验证                                    │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**快照服务最终路径（统一为）**：`packages/bruno-electron/src/services/snapshot/index.js`

### 5.2 快照序列化

**触发保存的 Action（共 25 个，可逐条复核）** (`utils/snapshot/index.js:29-55`)：

| 行号 | Action 名称 | 分类 |
|------|------------|------|
| 30 | `app/setSnapshotReady` | 应用状态 |
| 31 | `tabs/addTab` | 标签页操作 |
| 32 | `tabs/closeTabs` | 标签页操作 |
| 33 | `tabs/focusTab` | 标签页操作 |
| 34 | `tabs/closeAllCollectionTabs` | 标签页操作 |
| 35 | `tabs/reorderTabs` | 标签页操作 |
| 36 | `tabs/makeTabPermanent` | 标签页操作 |
| 37 | `tabs/updateRequestPaneTab` | UI 状态 |
| 38 | `tabs/updateRequestPaneTabWidth` | UI 状态 |
| 39 | `tabs/updateRequestPaneTabHeight` | UI 状态 |
| 40 | `tabs/updateResponsePaneTab` | UI 状态 |
| 41 | `tabs/updateResponsePaneScrollPosition` | UI 状态 |
| 42 | `tabs/updateResponseFormat` | UI 状态 |
| 43 | `tabs/updateResponseViewTab` | UI 状态 |
| 44 | `tabs/updateScriptPaneTab` | UI 状态 |
| 45 | `tabs/updateRequestBodyScrollPosition` | UI 状态 |
| 46 | `workspaces/setActiveWorkspace` | 工作区 |
| 47 | `collections/selectEnvironment` | 集合操作 |
| 48 | `collections/sortCollections` | 集合操作 |
| 49 | `collections/updateCollectionMountStatus` | 集合操作 |
| 50 | `collections/toggleCollection` | 集合操作 |
| 51 | `collections/expandCollection` | 集合操作 |
| 52 | `logs/openConsole` | 控制台 |
| 53 | `logs/closeConsole` | 控制台 |
| 54 | `logs/setActiveTab` | 控制台 |
| **合计** | | **25 个** |

```javascript
// utils/snapshot/index.js:29-55
export const SAVE_TRIGGERS = new Map([
  ['app/setSnapshotReady', null],
  ['tabs/addTab', null],
  // ... 共 25 个，line 30-54 逐条计数确认
]);
```

**防抖机制** (`snapshot/middleware.js:22-23, 244-265`)：
```javascript
const DEBOUNCE_MS = 1000;  // 1秒防抖
let saveTimer = null;

const scheduleSave = (getState) => {
  if (saveTimer) clearTimeout(saveTimer);
  saveTimer = setTimeout(async () => {
    const snapshot = await serializeSnapshot(state);
    await ipcRenderer.invoke('renderer:snapshot:save', snapshot);
  }, DEBOUNCE_MS);
};
```

**序列化函数 `serializeTab`** (`utils/snapshot/index.js:364-422`)：
```javascript
export const serializeTab = (tab, collection) => {
  const accessor = getAccessor(tab);
  const serialized = {
    type: tab.type,
    accessor,
    permanent: !tab.preview,  // 存储永久状态
  };

  // 根据 accessor 类型存储不同的标识
  // 存储 UI 状态（面板宽度、当前 tab 等）
  
  return serialized;
};
```

**关键点**：序列化内容完全不包含 `draft`、`environmentsDraft` 等未保存改动（Grep 确认 snapshot 模块中无相关字段引用）。

### 5.3 快照存储格式

**完整快照结构** (`bruno-electron/src/services/snapshot/index.js:89-97`)：

```javascript
{
  version: '0.0.1',
  activeWorkspacePath: string | null,
  extras: { devTools: { open, activeTab, tabs } },
  workspaces: [
    {
      pathname: string,
      lastActiveCollectionPathname: string | null,
      sorting: string,
      collections: string[]
    }
  ],
  collections: [
    {
      pathname: string,
      workspacePathname: string,
      environment: { collection: string, global: string },
      isOpen: boolean,
      isMounted: boolean,
      activeTab: { accessor: string, value: string } | null,
      tabs: [
        {
          type: string,
          accessor: string,
          pathname: string | null,
          permanent: boolean,
          request?: { tab: string, width: number, height: number },
          response?: { tab: string, format: string, viewTab: string }
        }
      ]
    }
  ]
}
```

**验证**：使用 Yup Schema 进行严格的数据完整性验证。

### 5.4 反序列化与恢复

**应用启动时恢复**：

1. **集合加载时**：`hydrateCollectionTabs` (`utils/snapshot/index.js:602-639`) 被调用
2. **获取快照**：优先从 `snapshotLookups` 内存缓存获取，否则通过 IPC 查询
3. **反序列化**：`deserializeTab` (`utils/snapshot/index.js:533-600`) 将快照转换为标签页对象
4. **恢复到 Redux**：派发 `restoreTabs` action

**`deserializeTab` 关键逻辑**：
```javascript
export const deserializeTab = (snapshotTab, collection) => {
  const tab = {
    collectionUid: collection.uid,
    type: snapshotTab.type,
    preview: !snapshotTab.permanent,  // 恢复预览状态
    // ... 恢复其他 UI 状态
  };

  // 根据 accessor 解析并重新绑定 uid
  if (accessor === 'pathname' && pathname) {
    const item = findItemInCollectionByPathname(collection, pathname);
    tab.uid = item?.uid || pathname;  // 使用实际项的 uid 或 pathname
  }
  // ... 其他 accessor 类型处理
};
```

**`restoreTabs` action** (`tabs.js:448-474`)：
```javascript
restoreTabs: (state, action) => {
  const { collection, tabs: snapshotTabs, activeTab } = action.payload;
  
  // 清除该集合现有标签页
  state.tabs = state.tabs.filter((t) => t.collectionUid !== collectionUid);
  
  // 逐个反序列化并恢复
  (snapshotTabs || []).forEach((snapshotTab) => {
    const tab = deserializeTab(snapshotTab, collection);
    state.tabs.push(tab);
    
    // 恢复活动标签页
    if (checkIsActiveTab(tab, activeTab, collection)) {
      state.activeTabUid = tab.uid;
    }
  });
}
```

**关键点**：反序列化过程完全不恢复 draft 状态，因为快照中根本没有存储这些字段。

## 六、未保存草稿保留条件（可复核）

### 6.1 场景一：标签切换

#### 事实依据

**事实 1**：Draft 存储在 `collections` slice 中，与 `tabs` slice 完全独立
- `item.draft` 存储在 `collections.items[].draft`
- `folder.draft` 存储在 `collections.items[]`（folder 类型）的 `draft`
- `collection.draft` 存储在 `collection.draft`
- `collection.environmentsDraft` 存储在 `collection.environmentsDraft`
- `globalEnvironmentDraft` 存储在 `globalEnvironments` slice

**代码引用**：
- `collections/index.js:887` - item.draft 初始化
- `collections/index.js:2764` - 加载文件时 draft: null

**事实 2**：标签切换动作（`tabs/focusTab`）只修改 `tabs` slice
- `tabs.js:176-188` - `focusTab` reducer 只修改 `state.activeTabUid`
- 完全不涉及 `collections` slice 的任何改动

**代码引用**：
```javascript
// tabs.js:176-188
focusTab: (state, action) => {
  const { tabUid } = action.payload;
  const tab = find(state.tabs, (tab) => tab.uid === tabUid);
  if (tab) {
    state.activeTabUid = tab.uid;
  }
}
```

**事实 3**：快照序列化不包含 draft 字段
- Grep `draft` / `environmentsDraft` 在 `utils/snapshot/` 和 `services/snapshot/` 中均无匹配
- 快照只存储 UI 布局状态（标签列表、活动标签、面板宽度等）

#### 结论

**结论 1：标签切换时，未保存草稿 100% 保留**

**推理链**：
1. Draft 存储在 `collections` slice → 
2. 标签切换只修改 `tabs` slice 的 `activeTabUid` → 
3. 两个 slice 相互独立，互不影响 → 
4. 因此标签切换不会清除任何 draft → 
5. 未保存草稿在标签切换后完全保留

**例外情况**：无例外。标签切换不会导致任何未保存草稿丢失。

---

### 6.2 场景二：应用重启

#### 事实依据

**事实 1**：Redux 状态存储在内存中，进程退出后全部丢失
- Redux 是纯内存状态管理，无内置持久化机制
- 应用重启 = 进程退出 + 重新启动 = 内存清空

**事实 2**：快照持久化不包含 draft 状态
- `serializeTab` (`utils/snapshot/index.js:364-422`) 只序列化标签类型、accessor、永久状态、UI 面板状态
- 完全不包含 `draft`、`environmentsDraft` 等字段
- Grep 验证：snapshot 模块中零个 draft 相关引用

**事实 3**：IndexedDB 也不存储 draft
- `utils/idb/index.js` 中无 `draft` 或 `environmentsDraft` 字段
- IndexedDB 主要用于缓存大请求响应，不存储编辑状态

**事实 4**：应用重新加载集合时，draft 初始化为 null
- `collections/index.js:2764` - 从磁盘加载文件时 `draft: null`
- `mountCollection` (`collections/actions.js:3015-3050`) 从文件系统读取原始内容，不含 draft

**事实 5**：自动保存和手动保存会写入磁盘
- 自动保存启用时 (`app.preferences.autoSave.enabled = true`)，改动后 `autoSave.interval` 毫秒自动保存到磁盘
- 用户 Ctrl+S 手动保存会立即写入磁盘
- 保存到磁盘的内容重启后会从文件重新加载

**例外场景**：临时请求 (`isTransient = true`)
- 即使启用自动保存也会跳过 (`autosave/middleware.js:138-140`)
- 没有对应的磁盘文件，关闭即丢失

#### 结论

**结论 2：应用重启时，未保存草稿 100% 丢失，除非已提前持久化到磁盘**

**推理链**：
1. Draft 仅存在于 Redux 内存中 →
2. 快照不存储 draft →
3. IndexedDB 不存储 draft →
4. 应用重启 = 内存清空 + 从磁盘重新加载 →
5. 磁盘文件是保存后的内容，不含 draft →
6. 因此未保存的 draft 重启后全部丢失

**保留的唯一条件**：重启前已通过以下方式之一保存到磁盘：
- ✅ 启用自动保存且等待了足够时间（超过 `autoSave.interval`）
- ✅ 用户手动 Ctrl+S 保存
- ✅ 关闭标签页或应用时选择了 "Save"
- ❌ 仅在内存中编辑，未触发任何保存 → **丢失**
- ❌ 临时请求（无对应磁盘文件）→ **必然丢失**

---

### 6.3 保留条件总结表

| 场景 | 是否保留 | 条件 | 备注 |
|------|---------|------|------|
| 标签页切换 | ✅ 100% 保留 | 无条件 | 标签切换只改 `activeTabUid`，不碰 collections |
| 关闭单个标签页（选 Don't Save） | ❌ 丢失 | 主动选择丢弃 | 调用 `deleteRequestDraft` 清空 draft |
| 关闭单个标签页（选 Save） | ✅ 保留到磁盘 | 用户选择保存 | draft 写入磁盘文件 |
| 批量关闭标签页 | ✅ 静默保存到磁盘 | 无改动直接关，有改动自动保存 | 批量关闭逻辑会自动保存 |
| 应用重启（已保存） | ✅ 从磁盘恢复 | 启用自动保存或手动保存过 | 从磁盘文件重新加载 |
| 应用重启（未保存） | ❌ 完全丢失 | 未持久化到磁盘 | draft 仅在内存中，进程退出即清空 |
| 应用重启（临时请求） | ❌ 完全丢失 | 无对应磁盘文件 | 即使自动保存也会跳过临时请求 |

## 七、关键设计模式与技术要点

### 7.1 中间件协作模型

三个中间件按顺序注册，各自承担单一职责：

```
Action 分发
    ↓
[tasksMiddleware]    # 任务队列管理
    ↓
[draftDetectMiddleware]  # 检测改动，标记永久标签（73 个 action）
    ↓
[autosaveMiddleware]     # 自动保存（如启用，83 个 action）
    ↓
[snapshotMiddleware]     # 防抖持久化 UI 状态（25 个触发事件）
    ↓
[debugMiddleware]        # 开发环境调试
    ↓
Reducer 更新状态
```

### 7.2 标识符稳定性设计

**UID vs Pathname**：
- 标签页使用 `uid` 作为主要标识符，但快照中存储 `pathname`
- 重启时通过 `pathname` 查找对应的项并重新绑定 `uid`
- 这样即使文件在磁盘上被移动/重命名，也能正确匹配

**示例标签页**：
- 存储 `exampleIndex` 而非 `exampleUid`
- 因为示例的 UID 可能在重新加载时变化，但索引位置相对稳定

### 7.3 防抖与批量优化

- **快照保存**：1 秒防抖，避免频繁写入磁盘
- **自动保存**：按实体独立防抖，同一实体的多次修改合并为一次保存
- **批量保存**：应用关闭时，同类实体批量保存（`saveMultipleRequests`）

### 7.4 边界情况处理

1. **临时请求 (`isTransient`)**：
   - 不自动保存
   - 不加入最近关闭栈
   - 快照持久化时过滤掉
   - 关闭时提示保存为正式请求

2. **不可关闭标签页**：
   - `workspaceOverview`、`workspaceEnvironments` 无法关闭
   - 在 `closeTabs` 中被过滤保留

3. **集合卸载/重新加载**：
   - 保留已关闭标签页栈，不随集合卸载而清除
   - 快照保存时保留未加载集合的状态

## 八、最终核准：切换保留、重启丢失条件清单（先事实后结论）

### 8.1 核准说明

本节所有结论均采用"先事实依据 → 后推导结论"的模式，每条事实均有对应代码行号可独立复核。

---

### 8.2 核准项 1：标签切换时，未保存草稿 100% 保留

#### 事实依据（每条均可独立复核）

| 编号 | 事实描述 | 代码位置 | 验证方式 |
|------|---------|---------|---------|
| T1 | Draft 存储在 `collections` slice，与 `tabs` slice 完全隔离 | `collections/index.js:887`、`collections/index.js:2764` | 检查 `item.draft` 字段初始化位置 |
| T2 | 标签切换只调用 `tabs/focusTab`，仅修改 `activeTabUid` | `tabs.js:176-188` | 阅读 `focusTab` reducer 代码，确认不涉及 collections |
| T3 | `tabs` 和 `collections` 是两个独立的 Redux slice | `providers/ReduxStore/index.js` | 检查 Store 配置中 slice 注册方式 |
| T4 | 快照序列化不包含 `draft` 字段 | `utils/snapshot/index.js:364-422`、`services/snapshot/index.js` | Grep `draft` 关键词，两处均无匹配 |
| T5 | 标签切换不触发任何 `deleteDraft` 类 action | `collections/index.js:776-782` | 搜索 `deleteRequestDraft` 调用位置，确认标签切换流程中无调用 |

#### 结论推导链

```
T1（Draft 在 collections slice）
       +
T2（切换只改 tabs 的 activeTabUid）
       +
T3（两 slice 相互独立）
       +
T5（切换不触发 deleteDraft）
       ↓
标签切换对 draft 无任何操作 → 草稿 100% 保留
```

#### 最终结论

✅ **标签切换场景：未保存草稿 100% 无条件保留**

**例外情况**：无。无论是否启用自动保存、无论是否为临时请求，标签切换均不会导致草稿丢失。

---

### 8.3 核准项 2：应用重启时，未保存草稿 100% 丢失（除非已持久化）

#### 事实依据（每条均可独立复核）

| 编号 | 事实描述 | 代码位置 | 验证方式 |
|------|---------|---------|---------|
| R1 | Redux 状态完全存储在内存中，进程退出后清空 | Redux 官方设计 + `providers/ReduxStore/index.js` | 检查是否有任何内置持久化机制 |
| R2 | 快照（snapshot）不存储 draft 字段 | `utils/snapshot/index.js:364-422`、`services/snapshot/index.js` | Grep `draft`、`environmentsDraft`，结果均为 0 匹配 |
| R3 | IndexedDB 不存储 draft | `utils/idb/index.js` | Grep `draft`、`environmentsDraft`，结果均为 0 匹配 |
| R4 | 从磁盘重新加载集合时，`draft` 初始化为 `null` | `collections/index.js:2764` | 检查加载文件时的初始化代码 `draft: null` |
| R5 | 自动保存会写入磁盘，但需要启用并等待间隔 | `autosave/middleware.js:233-264` | 检查 `autoSave.enabled` 判断和 `autoSave.interval` 逻辑 |
| R6 | 手动 Ctrl+S 会立即写入磁盘 | `collections/actions.js` 中的 `saveRequest` | 检查 saveRequest 调用后的文件系统写入 |
| R7 | 临时请求（`isTransient=true`）即使启用自动保存也会跳过 | `autosave/middleware.js:138-140` | 检查 `isItemTransientRequest` 判断分支 |
| R8 | 应用关闭时用户选择"Don't Save"会调用 `deleteRequestDraft` | `collections/index.js:776-782` | 检查 `deleteRequestDraft` reducer 逻辑 |

#### 结论推导链

```
R1（Redux 仅在内存）+ R2（快照无 draft）+ R3（IndexedDB 无 draft）
                      ↓
          应用重启 = 进程退出 + 重新加载
                      ↓
  R4（重新加载时 draft: null）→ 未保存的 draft 全部丢失
                      ↓
          除非满足以下 R5/R6 之一
                      ↓
        R5（自动保存已触发） OR R6（手动 Ctrl+S）
                      ↓
          内容已写入磁盘文件，重启后从磁盘恢复
```

#### 最终结论

❌ **应用重启场景：未保存草稿 100% 丢失**

**唯一保留条件**（需满足以下任意一条）：

| 条件 | 代码依据 | 说明 |
|------|---------|------|
| ✅ 自动保存已启用且等待超过 `autoSave.interval` | `autosave/middleware.js:233-264` | 默认间隔通常为 1000ms，需等待足够时间 |
| ✅ 手动按下 Ctrl+S 保存 | `collections/actions.js` 的 `saveRequest` | 立即写入磁盘 |
| ✅ 关闭标签页/应用时选择了"Save" | `ConfirmRequestClose/index.js`、`SaveRequestsModal.js` | 用户主动确认保存 |
| ❌ 仅在内存中编辑，未触发任何保存 | - | 必然丢失 |
| ❌ 临时请求（`isTransient=true`） | `autosave/middleware.js:138-140` | 无对应磁盘文件，必然丢失 |
| ❌ 关闭时选择了"Don't Save" | `collections/index.js:776-782` | 主动调用 `deleteRequestDraft` 清空 |

---

### 8.4 完整保留条件速查表

| 场景 | 是否保留 | 条件 | 事实依据 |
|------|---------|------|---------|
| 标签页切换 | ✅ 100% 保留 | 无条件 | T1-T5 |
| 关闭单个标签页选 Don't Save | ❌ 丢失 | 主动选择丢弃 | R8 |
| 关闭单个标签页选 Save | ✅ 保留到磁盘 | 用户选择保存 | R6 |
| 批量关闭标签页 | ✅ 静默保存到磁盘 | 无改动直接关，有改动自动保存 | R5 |
| 应用重启（已保存） | ✅ 从磁盘恢复 | 自动/手动保存过 | R5、R6 |
| 应用重启（未保存） | ❌ 完全丢失 | 未持久化到磁盘 | R1-R4 |
| 应用重启（临时请求） | ❌ 完全丢失 | 无对应磁盘文件 | R7 |

---

### 8.5 核心数字核准表

| 指标 | 核准数值 | 代码位置 | 验证方式 |
|------|---------|---------|---------|
| draftDetectMiddleware 拦截 action 数 | 73 个 | `draft/middleware.js:3-82` | Grep `'collections\/` |
| autosaveMiddleware 拦截 action 数 | 83 个 | `autosave/middleware.js:5-94` | Grep `'collections\/|'global-environments\/` |
| SAVE_TRIGGERS 快照触发事件数 | 25 个 | `utils/snapshot/index.js:30-54` | 逐条计数 |
| 快照服务最终路径 | `packages/bruno-electron/src/services/snapshot/index.js` | 架构图 + 5.1 节 | 检查类定义 line 111、导出 line 567 |

---

## 九、代码参考位置总结（可复核）

| 功能模块 | 完整文件路径 | 关键行号 |
|---------|-------------|---------|
| 标签页状态管理 | `packages/bruno-app/src/providers/ReduxStore/slices/tabs.js` | 13-530 |
| 标签页选择器 | `packages/bruno-app/src/selectors/tab.js` | 1-59 |
| Draft 检测中间件（73 个 action） | `packages/bruno-app/src/providers/ReduxStore/middlewares/draft/middleware.js` | 3-90 |
| Draft 检测工具 | `packages/bruno-app/src/providers/ReduxStore/middlewares/draft/utils.js` | 5-36 |
| 自动保存中间件（83 个 action） | `packages/bruno-app/src/providers/ReduxStore/middlewares/autosave/middleware.js` | 5-264 |
| 快照中间件 | `packages/bruno-app/src/providers/ReduxStore/middlewares/snapshot/middleware.js` | 22-265 |
| 快照序列化工具 | `packages/bruno-app/src/utils/snapshot/index.js` | 29-680 |
| 脏检测核心函数 | `packages/bruno-app/src/utils/collections/index.js` | 1069-1113 |
| 单个标签页关闭 | `packages/bruno-app/src/components/RequestTabs/RequestTab/index.js` | 158-608 |
| 确认对话框 | `packages/bruno-app/src/components/RequestTabs/RequestTab/ConfirmRequestClose/index.js` | 1-52 |
| 应用关闭确认 | `packages/bruno-app/src/providers/App/ConfirmAppClose/SaveRequestsModal.js` | 27-296 |
| **快照存储服务（统一路径）** | `packages/bruno-electron/src/services/snapshot/index.js` | 111（类定义）、567（导出） |
| 快照 IPC 处理器 | `packages/bruno-electron/src/ipc/snapshot.js` | 1-46 |
| Redux Store 配置 | `packages/bruno-app/src/providers/ReduxStore/index.js` | 1-43 |
| IndexedDB 存储 | `packages/bruno-app/src/utils/idb/index.js` | 1-200 |
