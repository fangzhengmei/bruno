# 标签页会话状态与未保存改动管理分析

## 一、整体架构概览

Bruno 使用 Redux Toolkit 作为状态管理库，通过三个核心中间件协同工作实现标签页会话管理：

```
Redux Store
├── slices
│   ├── tabs.js              # 标签页状态管理
│   ├── collections/index.js # 集合与条目状态管理（含draft）
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

**单个标签页对象** (`tabs.js:129-159`) 包含丰富的UI状态：

```javascript
{
  uid: string,
  collectionUid: string,
  type: 'http-request' | 'graphql-request' | 'grpc-request' | 'ws-request' | 'folder-settings' | 'collection-settings' | ...,
  pathname: string | null,           // 文件系统路径
  preview: boolean,                  // 是否为预览模式（点击打开vs双击打开）
  isTransient?: boolean,             // 是否为临时请求（无持久化文件）
  
  // UI布局状态
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

**响应示例标签页** 使用 `exampleIndex` 或 `exampleName` 进行额外匹配：
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

Bruno 采用 **Copy-on-Write** 模式管理未保存改动。每个可编辑对象（请求、文件夹、集合、环境）都有一个 `draft` 字段：

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

### 3.2 脏检测核心函数

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

### 3.3 各类实体的 Draft 状态

| 实体类型 | Draft 位置 | 检测方式 | 删除 Draft Action |
|---------|-----------|---------|------------------|
| 请求 | `item.draft` | `hasRequestChanges(item)` | `deleteRequestDraft` |
| 文件夹 | `folder.draft` | `folder.draft != null` | `deleteFolderDraft` |
| 集合 | `collection.draft` | `collection.draft != null` | `deleteCollectionDraft` |
| 集合环境 | `collection.environmentsDraft` | `collection.environmentsDraft != null` | `clearEnvironmentsDraft` |
| 全局环境 | `state.globalEnvironments.globalEnvironmentDraft` | `globalEnvironmentDraft != null` | `clearGlobalEnvironmentDraft` |

### 3.4 Draft 自动检测中间件

`draftDetectMiddleware` (`middlewares/draft/middleware.js:84-90`) 监听 80+ 种会产生改动的 action：

```javascript
const actionsToIntercept = [
  // Request-level actions (40+)
  'collections/requestUrlChanged',
  'collections/updateAuth',
  'collections/addQueryParam',
  // ... 更多请求级操作
  
  // Folder-level actions (15+)
  'collections/addFolderHeader',
  'collections/updateFolderVar',
  // ... 更多文件夹级操作
  
  // Collection-level actions (20+)
  'collections/addCollectionHeader',
  'collections/updateCollectionAuth',
  // ... 更多集合级操作
];

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

### 3.5 自动保存机制

`autosaveMiddleware` (`middlewares/autosave/middleware.js:233-264`) 在用户启用自动保存时工作：

**触发时机**：
- 监听与 draft 检测相同的 action 列表
- 自动保存启用时，为每个有改动的实体调度延迟保存
- 保存间隔由 `autoSave.interval` 配置控制

**去重机制**：
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

### 5.1 快照持久化架构

**三层持久化架构**：

```
┌─────────────────────────────────────────────────┐
│  Renderer Process (React + Redux)               │
│  ┌───────────────────────────────────────────┐  │
│  │  snapshotMiddleware                       │  │
│  │  - 监听 SAVE_TRIGGERS action              │  │
│  │  - 1秒防抖 serializeSnapshot()            │  │
│  │  - IPC: renderer:snapshot:save            │  │
│  └───────────────────────────────────────────┘  │
└───────────────────────────┬─────────────────────┘
                            │ IPC
┌───────────────────────────▼─────────────────────┐
│  Main Process (Electron)                        │
│  ┌───────────────────────────────────────────┐  │
│  │  SnapshotManager (electron-store)        │  │
│  │  - 存储到: ui-state-snapshot.json        │  │
│  │  - Yup Schema 验证                       │  │
│  │  - 查找缓存优化                          │  │
│  └───────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

### 5.2 快照序列化

**触发保存的 Action** (`utils/snapshot/index.js:29-55`)：
```javascript
export const SAVE_TRIGGERS = new Map([
  ['tabs/addTab', null],
  ['tabs/closeTabs', null],
  ['tabs/focusTab', null],
  ['tabs/makeTabPermanent', null],
  ['tabs/updateRequestPaneTab', null],
  ['tabs/updateRequestPaneTabWidth', null],
  ['tabs/updateResponsePaneTab', null],
  // ... 共 30+ 种触发事件
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
  const accessor = getAccessor(tab);  // 'pathname' | 'type' | 'pathname::exampleIndex'
  const serialized = {
    type: tab.type,
    accessor,
    permanent: !tab.preview,  // 存储永久状态
  };

  // 根据 accessor 类型存储不同的标识
  if (accessor === 'pathname') {
    const item = findItemInCollection(collection, tab.uid);
    serialized.pathname = item?.pathname || tab.pathname;
  } else if (accessor === 'pathname::exampleIndex') {
    // 存储 exampleIndex 而非 uid，因为 uid 可能变化
  }

  // 存储UI状态
  if (isRequest && tab.requestPaneTab !== undefined) {
    serialized.request = {
      tab: tab.requestPaneTab,
      width: tab.requestPaneWidth,
      height: tab.requestPaneHeight
    };
  }
  // 响应面板状态类似...
};
```

### 5.3 快照存储格式

**完整快照结构** (`services/snapshot/index.js:89-97`)：

```javascript
{
  version: '0.0.1',
  activeWorkspacePath: string | null,
  extras: {
    devTools: {
      open: boolean,
      activeTab: 'terminal' | 'console' | 'network' | 'performance',
      tabs: {}
    }
  },
  workspaces: [
    {
      pathname: string,
      lastActiveCollectionPathname: string | null,
      sorting: 'default' | 'alphabetical' | 'reverseAlphabetical',
      collections: string[]  // collection pathnames
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
          accessor: 'pathname' | 'type' | 'pathname::exampleIndex' | 'pathname::exampleName',
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

**Schema 验证**：使用 Yup 进行严格的 schema 验证，确保数据完整性。

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
    // ... 恢复其他UI状态
  };

  // 根据 accessor 解析并重新绑定 uid
  if (accessor === 'pathname' && pathname) {
    const item = findItemInCollectionByPathname(collection, pathname);
    tab.uid = item?.uid || pathname;  // 使用实际项的uid或pathname
  } else if (accessor === 'pathname::exampleIndex') {
    // 解析 exampleIndex 并查找对应的 example uid
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

### 5.5 工作区与多集合

**工作区隔离** (`utils/snapshot/index.js:128-244`)：
- 快照支持多工作区，每个工作区有独立的标签页状态
- 使用 `workspacePathname::collectionPathname` 作为复合键
- 支持集合在多个工作区间共享，此时每个工作区维护独立的标签页状态

**活动标签页匹配** (`utils/snapshot/index.js:459-488`)：
```javascript
export const isActiveTab = (tab, activeTab, collection) => {
  const { accessor, value } = activeTab;
  
  if (accessor === 'type') return tab.type === value;
  if (accessor === 'pathname') {
    const item = findItemInCollection(collection, tab.uid);
    return tab.type !== 'response-example' && 
           (item?.pathname === value || tab.pathname === value);
  }
  // ... 其他 accessor 类型
};
```

## 六、关键设计模式与技术要点

### 6.1 中间件协作模型

三个中间件按顺序注册，各自承担单一职责：

```
Action 分发
    ↓
[tasksMiddleware]    # 任务队列管理
    ↓
[draftDetectMiddleware]  # 检测改动，标记永久标签
    ↓
[autosaveMiddleware]     # 自动保存（如启用）
    ↓
[snapshotMiddleware]     # 防抖持久化UI状态
    ↓
[debugMiddleware]        # 开发环境调试
    ↓
Reducer 更新状态
```

### 6.2 标识符稳定性设计

**UID vs Pathname**：
- 标签页使用 `uid` 作为主要标识符，但快照中存储 `pathname`
- 重启时通过 `pathname` 查找对应的项并重新绑定 `uid`
- 这样即使文件在磁盘上被移动/重命名，也能正确匹配

**示例标签页**：
- 存储 `exampleIndex` 而非 `exampleUid`
- 因为示例的 UID 可能在重新加载时变化，但索引位置相对稳定

### 6.3 防抖与批量优化

- **快照保存**：1秒防抖，避免频繁写入磁盘
- **自动保存**：按实体独立防抖，同一实体的多次修改合并为一次保存
- **批量保存**：应用关闭时，同类实体批量保存（`saveMultipleRequests`）

### 6.4 边界情况处理

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

## 七、代码参考位置总结

| 功能模块 | 主要文件 | 关键行号 |
|---------|---------|---------|
| 标签页状态管理 | `providers/ReduxStore/slices/tabs.js` | 13-530 |
| 标签页选择器 | `selectors/tab.js` | 1-59 |
| Draft 检测中间件 | `providers/ReduxStore/middlewares/draft/middleware.js` | 1-90 |
| Draft 检测工具 | `providers/ReduxStore/middlewares/draft/utils.js` | 1-40 |
| 自动保存中间件 | `providers/ReduxStore/middlewares/autosave/middleware.js` | 1-264 |
| 快照中间件 | `providers/ReduxStore/middlewares/snapshot/middleware.js` | 1-303 |
| 快照序列化工具 | `utils/snapshot/index.js` | 1-680 |
| 脏检测函数 | `utils/collections/index.js` | 1069-1113 |
| 单个标签页关闭 | `components/RequestTabs/RequestTab/index.js` | 113-608 |
| 确认对话框 | `components/RequestTabs/RequestTab/ConfirmRequestClose/index.js` | 1-52 |
| 应用关闭确认 | `providers/App/ConfirmAppClose/SaveRequestsModal.js` | 1-296 |
| 快照存储服务 | `bruno-electron/src/services/snapshot/index.js` | 1-567 |
| Redux Store 配置 | `providers/ReduxStore/index.js` | 1-43 |
