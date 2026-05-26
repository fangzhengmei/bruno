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

它返回从 Collection 根到目标 Item 的完整路径数组，顺序为 **从根到叶**：

```
[collectionItem(虚拟根), folderA, folderB, requestItem]
```

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

  // 3. 以 Collection 级 auth 为兜底
  const collectionRoot = collection?.draft?.root || collection?.root || {};
  const collectionAuth = get(collectionRoot, 'request.auth', { mode: 'none' });
  let effectiveAuth = collectionAuth;

  // 4. 反向遍历文件夹（从近到远），找到第一个非 inherit/non-none 的 auth
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
  const collectionRoot = collection?.draft?.root || collection?.root || {};
  let collectionAuth = collectionRoot?.request?.auth || { mode: 'none' };
  let effectiveAuth = collectionAuth;

  // 正向遍历：从远到近，后面的覆盖前面的
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
    ├─ request.auth.mode === 'inherit' ?
    │      │
    │      ├─ YES → 从 Collection 根取 auth 为 effectiveAuth
    │      │         遍历树路径上的 Folder（从远到近）：
    │      │           若 Folder.auth.mode ∉ {'inherit', 'none'}
    │      │             → effectiveAuth = Folder.auth（覆盖）
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

## 七、总结：核心规则速查

1. **三级定义**：Collection → Folder → Request，每级都可独立设置 auth
2. **inherit = 向上找**：`mode: 'inherit'` 触发沿树向上查找第一个非 `inherit`/`none` 的祖先 auth
3. **就近覆盖**：多个 Folder 都设了具体 auth 时，离请求最近的生效
4. **none 不传递**：Folder 设为 `'none'` 不会被子请求继承，子请求 `inherit` 会跳过它继续向上找
5. **请求自主**：请求自身非 `inherit` 时，祖先 auth 完全不影响
6. **draft 参与**：未保存的 draft auth 在 UI 层也会参与继承计算
7. **整体替换**：auth 继承是整个对象的替换，不是字段级 merge
8. **两段冗余**：`setAuthHeaders` 的第一段代码在正常流程下不会触发，是历史遗留的防御性代码