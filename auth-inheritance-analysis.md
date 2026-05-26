# Bruno 鉴权继承链分析

## 一、三层鉴权定义模型

Bruno 的鉴权（Auth）可以在三个层级上定义，形成一条从根到叶子的继承链：

| 层级 | 存储位置 | 数据结构路径 | 格式 |
|------|----------|-------------|------|
| Collection（集合根） | `collection.bru` 或 `opencollection.yml` | `collection.root.request.auth` | 同 Folder |
| Folder（文件夹） | `folder.bru` 或 `folder.yml` | `folder.root.request.auth` | 同 Collection |
| Request（请求） | `{name}.bru` 或 `{name}.yml` | `item.request.auth` | 独立 request schema |

三级 schema 均使用同一个 `authSchema`（定义于 `bruno-schema/src/collections/index.js:402`），其 `mode` 字段支持以下值：

```
'inherit' | 'none' | 'awsv4' | 'basic' | 'bearer' | 'digest' | 'ntlm' | 'oauth1' | 'oauth2' | 'wsse' | 'apikey'
```

其中 `'inherit'` 是继承机制的语义核心，`'none'` 表示显式禁用鉴权，其余为具体鉴权类型。

---

## 二、继承链的解析规则

### 2.1 树路径的构成

`getTreePathFromCollectionToItem(collection, item)` 定义于：
- CLI 层：`bruno-cli/src/utils/collection.js:439`
- App 层：`bruno-app/src/utils/collections/index.js`

实现核心：

```js
const getTreePathFromCollectionToItem = (collection, _item) => {
  let path = [];
  let item = findItemInCollection(collection, _item.pathname);
  while (item) {
    path.unshift(item);
    item = findParentItemInCollection(collection, item.pathname);
  }
  return path;
};
```

`findItemInCollection` 和 `findParentItemInCollection` 均在 `collection.items` 的扁平化结构中查找，**collection 自身不在 `collection.items` 中**，因此路径不包含集合虚拟根节点。

它返回从 **顶级 Item** 到目标 Item 的路径数组，顺序为 **从根到叶**：

```
[folderA, folderB, requestItem]   // 嵌套请求
[requestItem]                     // 顶级请求
```

**关键点**：路径始终包含目标 Item 自身（最后一个元素），遍历时通过 `i.type === 'folder'` 过滤，只处理 Folder 节点。Collection 级 auth 独立于树路径，在函数外部单独从 `collection.root.request.auth` 获取。

### 2.2 UI 层解析（`resolveInheritedAuth`）

位置：`bruno-app/src/utils/auth/index.js:7`

```js
export const resolveInheritedAuth = (item, collection) => {
  // 1. 合并 draft 和正式 request
  const mergedRequest = {
    ...(item.request || {}),
    ...(item.draft?.request || {})
  };

  // 2. 若非 inherit，直接返回
  if (!authMode || authMode !== 'inherit') {
    return mergedRequest;
  }

  // 3. 以 Collection 级 auth 为兜底（独立于树路径，从 collection.root 获取）
  const collectionRoot = collection?.draft?.root || collection?.root || {};
  const collectionAuth = get(collectionRoot, 'request.auth', { mode: 'none' });
  let effectiveAuth = collectionAuth;

  // 4. 树路径不含 collection，从顶级 item 到目标 item；
  //    反向遍历（从近到远），跳过目标 item 自身（i.type !== 'folder'），
  //    找到第一个非 inherit/non-none 的 Folder auth
  for (let i of [...requestTreePath].reverse()) {
    if (i.type === 'folder') {
      const folderAuth = i?.draft
        ? get(i, 'draft.request.auth')
        : get(i, 'root.request.auth');
      if (folderAuth && folderAuth.mode
          && folderAuth.mode !== 'none'
          && folderAuth.mode !== 'inherit') {
        effectiveAuth = folderAuth;
        break;  // 找到最近的即停止
      }
    }
  }

  return { ...mergedRequest, auth: effectiveAuth };
};
```

**优先级（从高到低）**：
1. **请求自身** 非 `inherit` 的 auth → 直接使用
2. **最近的祖先 Folder** 非 `inherit`/`none` 的 auth → 覆盖
3. **Collection 根** auth → 兜底
4. 默认 `{ mode: 'none' }` → 无鉴权

### 2.3 运行时解析（`mergeAuth`，CLI/Electron 层）

位置：`bruno-cli/src/utils/collection.js:449`

```js
const mergeAuth = (collection, request, requestTreePath) => {
  // Collection 级 auth 独立于树路径，从 collection.root 获取
  const collectionRoot = collection?.draft?.root || collection?.root || {};
  let collectionAuth = collectionRoot?.request?.auth || { mode: 'none' };
  let effectiveAuth = collectionAuth;

  // 树路径不含 collection，从顶级 item 到目标 item；
  // 正向遍历（从远到近），跳过目标 item 自身（i.type !== 'folder'），
  // 更近的 Folder auth 覆盖更远的
  for (let i of requestTreePath) {
    if (i.type === 'folder') {
      const folderRoot = i?.draft || i?.root;
      const folderAuth = get(folderRoot, 'request.auth');
      if (folderAuth && folderAuth.mode
          && folderAuth.mode !== 'none'
          && folderAuth.mode !== 'inherit') {
        effectiveAuth = folderAuth;  // 不 break，继续用更近的覆盖
      }
    }
  }

  // 仅当 request.auth.mode === 'inherit' 时才替换
  if (request.auth && request.auth.mode === 'inherit') {
    request.auth = effectiveAuth;
  }
};
```

与 UI 层逻辑等价：正向遍历不 break → 更近的 Folder 覆盖更远的 → 结果与反向 + break 一致。

**关键约束**：`mergeAuth` 只在 `request.auth.mode === 'inherit'` 时才做替换。如果请求显式设为 `'none'` 或具体类型，不会被祖先覆盖。

**Collection auth 的独立性**：Collection auth 不在树路径中，始终作为兜底值存在。当树路径中的所有 Folder 均为 `inherit` 或 `none` 时，最终使用的就是 Collection auth。

---

## 三、运行时合并的完整调用链

### 3.1 Electron（桌面端）执行流程

位置：`bruno-electron/src/ipc/network/prepare-request.js:354`

```
prepareRequest(item, collection)
  ├─ getTreePathFromCollectionToItem(collection, item)
  ├─ mergeHeaders(collection, request, requestTreePath, ...)
  ├─ mergeScripts(collection, request, requestTreePath, scriptFlow)
  ├─ mergeVars(collection, request, requestTreePath)
  ├─ mergeAuth(collection, request, requestTreePath)     ← auth 在此解析
  └─ setAuthHeaders(axiosRequest, request, collectionRoot) ← auth 在此注入请求
```

### 3.2 `setAuthHeaders` 的两段式处理

位置：`bruno-electron/src/ipc/network/prepare-request.js:11`

`setAuthHeaders` 是 auth 注入的最终执行函数，其结构为 **两段 if**：

**第一段**（`prepare-request.js:12-179`）：
```js
const collectionAuth = get(collectionRoot, 'request.auth');
if (collectionAuth && request.auth.mode === 'inherit') {
  switch (collectionAuth.mode) {
    case 'basic':   axiosRequest.basicAuth = ...; break;
    case 'bearer':  axiosRequest.headers['Authorization'] = ...; break;
    // ... 其他类型
  }
}
```

这段仅在 `request.auth.mode === 'inherit'` 时生效，使用 **Collection 根 auth** 注入。

**第二段**（`prepare-request.js:182-348`）：
```js
if (request.auth) {
  switch (request.auth.mode) {
    case 'basic':   axiosRequest.basicAuth = ...; break;
    case 'bearer':  axiosRequest.headers['Authorization'] = ...; break;
    // ... 其他类型
  }
}
```

这段对 `request.auth` 的任何 mode（包括继承解析后的具体类型）都生效。

**但注意**：由于 `mergeAuth` 在 `setAuthHeaders` 之前执行，它已经将 `'inherit'` 替换为了实际的 auth 对象。因此在正常流程下：
- 第一段条件 `request.auth.mode === 'inherit'` **永远不满足**
- 第二段始终执行最终的具体鉴权注入

第一段本质上是一个 **防御性兜底**，仅在 `mergeAuth` 未被调用或未能解析的极端情况下才会生效（此时只使用 Collection 根的 auth，忽略 Folder 层）。

### 3.3 覆盖优先级的实际效果

以一个三层嵌套为例：

```
Collection  auth: { mode: 'bearer', bearer: { token: 'COL' } }
  └─ FolderA  auth: { mode: 'basic', basic: { username: 'u1', password: 'p1' } }
       └─ FolderB  auth: { mode: 'inherit' }
            └─ Request  auth: { mode: 'inherit' }
```

解析过程：
1. `effectiveAuth` 初始 = `{ mode: 'bearer', ... }`（Collection 兜底）
2. 遍历 FolderA → `mode: 'basic'` ≠ inherit/non-none → `effectiveAuth = FolderA.basic`
3. 遍历 FolderB → `mode: 'inherit'` → 跳过
4. Request → `mode: 'inherit'` → 被替换为 `effectiveAuth = FolderA.basic`

**最终结果**：Request 使用 FolderA 的 Basic Auth。

再看一个 Folder auth 为 `'none'` 的例子：

```
Collection  auth: { mode: 'bearer', bearer: { token: 'COL' } }
  └─ FolderA  auth: { mode: 'none' }
       └─ Request  auth: { mode: 'inherit' }
```

解析过程：
1. `effectiveAuth` 初始 = `{ mode: 'bearer', ... }`
2. 遍历 FolderA → `mode: 'none'` → 跳过（`'none'` 不参与覆盖）
3. Request → `mode: 'inherit'` → 被替换为 `effectiveAuth = Collection.bearer`

**最终结果**：Request 使用 Collection 的 Bearer Token。`'none'` 不会传递下来，它只表示"此 Folder 不设置鉴权"，不阻断祖先 auth 的传递。

---

## 四、容易误解的要点

### 4.1 `'inherit'` vs `'none'` 的语义区别

| mode | 含义 | 对继承链的影响 |
|------|------|---------------|
| `'inherit'` | "我不设置，用上层的" | 继续向上查找，参与继承传递 |
| `'none'` | "我显式不需要鉴权" | **跳过**，不参与覆盖，但也不阻断祖先向更远后代的传递 |
| 具体类型 | "我用这种鉴权" | 成为当前后代的有效鉴权 |

**易错点**：很多用户以为 Folder 设为 `'none'` 就能让子请求不鉴权。实际上 `'none'` 在 `mergeAuth` 的判断中被显式排除了（条件 `mode !== 'none' && mode !== 'inherit'`），子请求如果是 `'inherit'`，会跳过 `'none'` 的 Folder，继续使用 Collection 根或更上层 Folder 的 auth。

**要让子请求真正不鉴权，子请求自身必须设为 `'none'`**。

### 4.2 Folder auth 的"就近原则"

当多个 Folder 都定义了非 inherit/non-none 的 auth 时，**离请求最近的那个生效**。这在 CLI 实现中是通过正向遍历 + 不 break 实现的（后面覆盖前面），在 UI 实现中是反向遍历 + break 实现的（找到第一个就停止）。两种方式结果一致。

### 4.3 Draft 与正式值的合并

`resolveInheritedAuth` 中对 Folder auth 的取值逻辑：
```js
const folderAuth = i?.draft
  ? get(i, 'draft.request.auth')    // 有 draft 优先用 draft
  : get(i, 'root.request.auth');    // 否则用 root（已保存的值）
```

这意味着 **未保存的 Folder draft 也会参与继承解析**。如果用户在 Folder 设置中编辑了 auth 但未保存，该 draft 值会立即影响子请求在 UI 中的显示。

### 4.4 `setAuthHeaders` 中的双重路径

如 3.2 节所述，`setAuthHeaders` 有两段逻辑：
- 第一段：检查 `request.auth.mode === 'inherit'` + Collection 根 auth → 几乎不会触发（因 mergeAuth 已替换）
- 第二段：检查 `request.auth` 的实际 mode → 正常执行

这是一段 **历史遗留的冗余代码**。第一段保留的唯一意义是在 `mergeAuth` 因某种原因未执行时提供兜底，但此时它只能读取 Collection 根 auth，无法识别 Folder 层的覆盖，可能产生与预期不一致的结果。

### 4.5 YML 格式与 BRU 格式的等价性

两种格式在 auth 继承上行为一致，因为：
- 解析后都映射到同一套 JSON 结构（`root.request.auth` / `request.auth`）
- `mergeAuth` 和 `resolveInheritedAuth` 操作的都是解析后的 JSON 对象
- `bruno-filestore/src/formats/yml/common/auth.ts` 中的 `toOpenCollectionAuth` / `toBrunoAuth` 仅做字段名转换，不改变 mode 语义

唯一的格式差异是序列化时的字段名不同（BRU 用 snake_case 如 `consumer_key`，YML 用 camelCase 如 `consumerKey`），但这不影响继承逻辑。

### 4.6 OAuth2 的 `autoFetchToken` / `autoRefreshToken` 默认值

在 `collectionBruToJson.js` 的 `authOAuth2` 语义动作中，各 grantType 有不同的默认值：

| grantType | autoFetchToken 默认 | autoRefreshToken 默认 |
|-----------|---------------------|----------------------|
| `password` | `true` | `false` |
| `authorization_code` | `true` | `false` |
| `implicit` | `true` | 不适用（无 refresh） |
| `client_credentials` | `true` | `false` |

这些默认值在序列化时通过 `safeParseJson(...) ?? defaultValue` 实现。如果用户在 Bru 文件中不写这些字段，解析后会填充默认值。**继承链上的 OAuth2 auth 在各层级独立存储完整配置，不会做字段级别的 merge**——继承是整个 auth 对象的替换，不是部分字段的合并。

---

## 五、继承链决策树

```
请求准备执行
    │
    ├─ getTreePathFromCollectionToItem(collection, item)
    │    → 返回 [顶级folder, ..., 父folder, requestItem]
    │    → 注意：collection 自身不在路径中
    │
    ├─ request.auth.mode === 'inherit' ?
    │      │
    │      ├─ YES → effectiveAuth = collection.root.request.auth（独立兜底）
    │      │         遍历树路径（从远到近 / 从近到远皆可）：
    │      │           跳过 requestItem 自身（i.type !== 'folder'）
    │      │           若 Folder.auth.mode ∉ {'inherit', 'none'}
    │      │             → effectiveAuth = Folder.auth（更近的覆盖更远的）
    │      │         最终 request.auth = effectiveAuth
    │      │
    │      └─ NO  → request.auth 保持不变（无论是 'none' 还是具体类型）
    │
    └─ setAuthHeaders 根据 request.auth.mode 的具体类型注入请求
```

---

## 六、测试覆盖验证

`bruno-app/src/utils/auth/index.spec.js` 中的测试用例验证了核心场景：

1. **最近 Folder 覆盖 Collection** → `expect(resolved.auth.mode).toBe('basic')`
2. **Folder 为 inherit 时回落到 Collection** → `expect(resolved.auth.mode).toBe('bearer')`
3. **请求自身非 inherit 时不继承** → `expect(resolved.auth.basic.username).toBe('override')`

`tests/auth/auth-mode-switch.spec.ts` 验证了切换 auth mode 时已保存的凭证不会丢失，这是 UI 层的草稿状态管理，与继承逻辑正交。

---

## 八、展示层与执行层的判定差异

代码中存在多份继承逻辑的实现，它们在 `'none'` 处理和 draft 读取上存在细微但重要的差异。

### 8.1 `inherit/none` 处理分歧

全代码库共有 7 处实现了继承逻辑，分为两大阵营：

| 实现位置 | 判定条件 | 是否排除 `'none'` | 是否考虑 draft |
|----------|----------|------------------|----------------|
| **执行层** | | | |
| `bruno-cli/src/utils/collection.js:458` | `mode !== 'none' && mode !== 'inherit'` | ✅ 是 | ✅ 是（`i?.draft \|\| i?.root`） |
| `bruno-electron/src/utils/collection.js:795` | `mode !== 'none' && mode !== 'inherit'` | ✅ 是 | ✅ 是 |
| `bruno-app/src/utils/auth/index.js:32` | `mode !== 'none' && mode !== 'inherit'` | ✅ 是 | ✅ 是 |
| **展示层（不一致的）** | | | |
| `bruno-app/src/components/RequestPane/Auth/index.js:60` | `mode !== 'inherit'` | ❌ 否 | ❌ 否（只读 `i.root`） |
| `bruno-app/src/components/FolderSettings/Auth/index.js:76` | `mode !== 'inherit'` | ❌ 否 | ✅ 是（`parentFolder?.draft \|\| ...`） |
| **展示层（一致的）** | | | |
| `bruno-app/src/components/ResponsePane/Timeline/index.js:34` | `mode !== 'none' && mode !== 'inherit'` | ✅ 是 | ❌ 否（只读 `i.root`） |
| `bruno-app/src/components/RequestPane/WSRequestPane/WSAuth/index.js:54` | `mode !== 'none' && mode !== 'inherit'` | ✅ 是 | ❌ 否（只读 `i.root`） |
| `bruno-app/src/components/RequestPane/GrpcRequestPane/GrpcAuth/index.js:63` | `mode !== 'none' && mode !== 'inherit'` | ✅ 是 | ❌ 否（只读 `i.root`） |

**分歧 1：`'none'` 是否参与继承**

`RequestPane/Auth/index.js:60` 和 `FolderSettings/Auth/index.js:76` 的判断条件只排除了 `'inherit'`，没有排除 `'none'`。这意味着：

```
Collection  auth: { mode: 'bearer', bearer: { token: 'COL' } }
  └─ FolderA  auth: { mode: 'none' }
       └─ Request  auth: { mode: 'inherit' }
```

- **展示层（RequestPane Auth 标签页）**：显示 "Auth inherited from FolderA: No Auth"
- **执行层（实际发送请求）**：跳过 FolderA 的 `'none'`，使用 Collection 的 bearer token

**这是一个功能性 Bug**：UI 告诉用户"此请求不鉴权"，但实际发送时却带了鉴权头。

**分歧 2：Folder `'none'` 与 Request `'none'` 的语义差异**

在执行层，两者的语义完全不同：
- **Folder auth = `'none'`**：仅表示"此 Folder 不提供鉴权配置"，不阻断继承链，也不会传递给后代
- **Request auth = `'none'`**：表示"此请求明确不使用任何鉴权"，`mergeAuth` 的 `request.auth.mode === 'inherit'` 条件不满足，直接跳过继承

但在 `RequestPane/Auth` 的展示逻辑中，Folder 的 `'none'` 被当作有效继承源，混淆了两者的语义。

### 8.2 未保存配置（draft）参与继承时的偏差

draft 是 Bruno 的核心特性——用户编辑但未保存的内容会进入 `item.draft` 字段，不影响 `item.root` 或 `item.request` 的已保存值。

**偏差来源 1：展示层只读 `root`，不读 `draft`**

`RequestPane/Auth/index.js:59`：
```js
const folderAuth = get(i, 'root.request.auth');  // 只读取已保存值
```

`ResponsePane/Timeline/index.js:33`：
```js
const folderAuth = get(i, 'root.request.auth');  // 只读取已保存值
```

而 `mergeAuth`（`bruno-cli/src/utils/collection.js:455`）：
```js
const folderRoot = i?.draft || i?.root;  // draft 优先
const folderAuth = get(folderRoot, 'request.auth');
```

场景：
1. 用户在 FolderA 中把 auth 从 `'none'` 改成 `'bearer'`，**未保存**（存在 `FolderA.draft.root.request.auth`）
2. 子请求 Request 的 auth 是 `'inherit'`
3. **展示层**：显示继承自 Collection（因为只读 `root`，看不到 draft）
4. **执行层（Electron 中点击发送）**：实际使用 FolderA 的 draft bearer token

**偏差来源 2：CLI 从不读取 draft**

CLI 直接从文件系统解析，`collection.items` 中没有 `draft` 字段。因此：
- CLI 执行：始终使用已保存的值
- Electron 执行：当前用户未保存的 draft 会影响执行结果

这意味着 **同样的请求，在 CLI 和 GUI 中执行结果可能不同**——如果用户修改了 Folder auth 但未保存，GUI 用新值，CLI 用旧值。

**偏差来源 3：Collection 级 draft 的读取不一致**

`prepareRequest` 第 357 行：
```js
const collectionRoot = collection?.draft?.root ? get(collection, 'draft.root', {}) : get(collection, 'root', {});
```

Collection 级的 draft 会影响执行，但 Folder 级 draft 的读取取决于具体实现。

### 8.3 如何核对实际生效的鉴权来源

当怀疑 UI 展示与实际执行不一致时，可通过以下方式核对：

**方法 1：查看 Response Timeline**

`ResponsePane/Timeline/index.js` 中的 `getEffectiveAuthSource` 在 `'none'` 处理上与执行层一致（排除 `'none'`），是最可靠的 UI 来源。它会显示：
- "Auth inherited from Collection" / "Auth inherited from FolderX"
- 配合 OAuth2 调用记录，可以看到实际 token 的获取来源

**方法 2：检查实际发送的请求头**

在 Timeline 的 Request 详情中查看 `Authorization` 等鉴权头的实际值，这是最直接的证据。

**方法 3：使用 CLI 执行验证**

```bash
bruno run path/to/request.bru --env production
```

CLI 不涉及 draft，结果代表"已保存配置的真实行为"。

**方法 4：代码级核对**

执行层的唯一真相来源是 `mergeAuth` + `setAuthHeaders` 第二段的组合，可在 `bruno-electron/src/ipc/network/prepare-request.js:376` 打断点查看 `request.auth` 被替换后的值。

---

## 九、总结：核心规则速查

1. **三级定义**：Collection → Folder → Request，每级都可独立设置 auth
2. **树路径不含 Collection**：`getTreePathFromCollectionToItem` 返回从顶级 Item 到目标 Item 的路径，不包含 Collection 虚拟根节点；Collection auth 通过 `collection.root.request.auth` 独立获取
3. **inherit = 向上找**：`mode: 'inherit'` 触发沿树向上查找第一个非 `inherit`/`none` 的祖先 Folder auth，若找不到则使用 Collection auth 兜底
4. **就近覆盖**：多个 Folder 都设了具体 auth 时，离请求最近的生效
5. **none 不传递（执行层）**：Folder 设为 `'none'` 不会被子请求继承，子请求 `inherit` 会跳过它继续向上找
6. **none 与 none 语义不同**：Folder auth = `'none'` 表示"此 Folder 不提供鉴权配置"；Request auth = `'none'` 表示"此请求明确不使用任何鉴权"
7. **请求自主**：请求自身非 `inherit` 时，祖先 auth 完全不影响
8. **draft 偏差**：执行层（Electron 发送请求）会考虑 Folder draft auth，但展示层（RequestPane Auth 标签页）只读已保存的 `root`，可能出现 UI 展示与实际执行不一致
9. **CLI/GUI 差异**：CLI 从文件解析无 draft，始终使用已保存值；GUI 未保存的 draft 会影响执行结果
10. **UI Bug 提示**：`RequestPane/Auth/index.js` 和 `FolderSettings/Auth/index.js` 未排除 Folder auth = `'none'`，可能误导用户
11. **整体替换**：auth 继承是整个对象的替换，不是字段级 merge
12. **两段冗余**：`setAuthHeaders` 的第一段代码在正常流程下不会触发，是历史遗留的防御性代码
13. **核对真相**：对鉴权来源存疑时，查看 Response Timeline 或实际请求头，而非 Auth 标签页的继承来源文字