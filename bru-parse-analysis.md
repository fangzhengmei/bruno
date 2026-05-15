# Bruno .bru 文件解析流程分析

## 概述

本文档详细分析 Bruno API Client 中 `.bru` 格式文件解析为内部 collection 对象的完整流程。整个解析过程分为三个主要层次：

1. **语法解析层** - `bruToJsonV2`：将 bru 文本格式解析为原始 JSON 对象
2. **结构转换层** - `parseBruRequest`：将原始 JSON 转换为标准化内部对象结构
3. **Schema 验证层** - `collectionSchema`：验证最终对象结构的完整性

---

## 第一层：语法解析 (bruToJsonV2)

### 核心实现文件

- `packages/bruno-lang/v2/src/bruToJson.js`

### 技术栈

- **Ohm-js**：声明式语法解析器，用于定义 bru 格式语法并生成 AST
- **Lodash**：对象合并、映射等操作

### 核心流程

#### 1. Grammar 语法定义

bru 文件由多个块（Blocks）组成：

```javascript
BruFile = (meta | http | grpc | ws | query | params | headers | metadata | auths | bodies | varsandassert | script | tests | settings | docs | example)*
```

**块类型分类**：

| 块类型 | 示例 | 说明 |
|--------|------|------|
| **Dictionary Blocks** | `headers { }` | 键值对结构，支持注释、禁用标记 |
| **Text Blocks** | `body:json { }` | 纯文本内容块，支持多行缩进 |
| **List Blocks** | `tags [ ]` | 列表结构 |

#### 2. Ohm Semantics 语义处理

```javascript
const sem = grammar.createSemantics().addAttribute('ast', {
  // 根节点：合并所有块
  BruFile(tags) { return _.reduce(tags.ast, (result, item) => _.mergeWith(result, item, concatArrays), {}); },
  
  // Dictionary 处理
  dictionary(_1, _2, _3, pairlist, _4) { return pairlist.ast; },
  
  // 键值对处理
  pair(_1, annotations, _2, key, _3, _4, _5, value, _6) {
    let res = {};
    res[key.ast] = Array.isArray(value.ast) ? value.ast : (value.ast ? value.ast.trim() : '');
    if (annotations.ast?.length) res[ANNOTATIONS_KEY] = annotations.ast;
    return res;
  },
  
  // HTTP 方法处理
  get(_1, dictionary) { return { http: { method: 'get', ...mapPairListToKeyValPair(dictionary.ast) }}; },
  post(_1, dictionary) { return { http: { method: 'post', ...mapPairListToKeyValPair(dictionary.ast) }}; },
  // ... 其他 HTTP 方法
})
```

#### 3. 辅助映射函数

| 函数名 | 功能 | 输出结构 |
|--------|------|----------|
| `mapPairListToKeyValPair` | 字典转对象 | `{ key: value }` |
| `mapPairListToKeyValPairs` | 字典转键值对数组 | `[{ name, value, enabled, annotations }]` |
| `mapRequestParams` | 请求参数映射 | `[{ name, value, enabled, type }]` |
| `mapPairListToKeyValPairsMultipart` | multipart 表单映射 | `[{ name, value, type, contentType }]` |
| `mapPairListToKeyValPairsFile` | 文件映射 | `[{ filePath, contentType, selected }]` |

#### 4. 特殊块处理

**认证块 (Auth Blocks)**：

- `auth:awsv4` → `{ auth: { awsv4: { accessKeyId, secretAccessKey, ... }}}`
- `auth:basic` → `{ auth: { basic: { username, password }}}`
- `auth:bearer`, `auth:digest`, `auth:ntlm`, `auth:oauth1`, `auth:oauth2`, `auth:wsse`, `auth:apikey`

**OAuth2 额外参数块**：

- `auth:oauth2:additional_params:auth_req:headers` → `oauth2_additional_parameters_auth_req_headers`
- `auth:oauth2:additional_params:access_token_req:headers/queryparams/body`
- `auth:oauth2:additional_params:refresh_token_req:headers/queryparams/body`

**Vars 和 Assert 块**：

- `vars:pre-request` → `{ vars: { req: [...] }}`
- `vars:post-response` → `{ vars: { res: [...] }}`
- `assert` → `{ assertions: [...] }`

**示例块 (Example)**：

- `example { }` → 调用 `parseExampleContent` 解析，返回 `{ examples: [parsedExample] }`

### 解析后的原始 JSON 结构示例

```javascript
{
  meta: { name: 'Send Bulk SMS', type: 'http', seq: 1, tags: ['foo', 'bar'] },
  http: { method: 'get', url: 'https://api.textlocal.in/send/:id', body: 'json', auth: 'bearer' },
  params: [{ name: 'apiKey', value: 'secret', enabled: true, type: 'query' }, ...],
  headers: [{ name: 'content-type', value: 'application/json', enabled: true }, ...],
  auth: {
    awsv4: { ... },
    basic: { ... },
    bearer: { token: '123' },
    oauth2: { grantType: 'authorization_code', ... }
  },
  body: {
    json: '{ "hello": "world" }',
    formUrlEncoded: [...],
    multipartForm: [...]
  },
  vars: { req: [...], res: [...] },
  assertions: [...],
  script: { req: '...', res: '...' },
  tests: '...',
  docs: '...',
  examples: [...]
}
```

---

## 第二层：结构转换 (parseBruRequest)

### 核心实现文件

- `packages/bruno-filestore/src/formats/bru/index.ts`

### 转换流程

#### 1. 请求类型确定

```javascript
let requestType = _.get(json, 'meta.type');
switch (requestType) {
  case 'http': requestType = 'http-request'; break;
  case 'graphql': requestType = 'graphql-request'; break;
  case 'grpc': requestType = 'grpc-request'; break;
  case 'ws': requestType = 'ws-request'; break;
  default: requestType = 'http-request';
}
```

#### 2. 基础字段提取与转换

```javascript
const transformedJson = {
  type: requestType,
  name: _.get(json, 'meta.name'),
  seq: !_.isNaN(sequence) ? Number(sequence) : 1,
  settings: _.get(json, 'settings', {}),
  tags: Array.isArray(tags) ? tags : [],
  request: {
    method: requestType === 'grpc-request' 
      ? _.get(json, 'grpc.method', '') 
      : String(_.get(json, 'http.method') ?? '').toUpperCase(),
    url: _.get(json, urlPath[requestType], _.get(json, urlPath.default)),
    headers: requestType === 'grpc-request' ? _.get(json, 'metadata', []) : _.get(json, 'headers', []),
    auth: _.get(json, 'auth', {}),
    body: _.get(json, 'body', {}),
    script: _.get(json, 'script', {}),
    vars: _.get(json, 'vars', {}),
    assertions: _.get(json, 'assertions', []),
    tests: _.get(json, 'tests', ''),
    docs: _.get(json, 'docs', '')
  },
  examples: _.get(json, 'examples', []).map(e => bruExampleToJson(e, true, requestType, _.get(json, 'http.method')))
};
```

#### 3. 请求类型特定字段处理

**gRPC 请求**：

```javascript
if (requestType === 'grpc-request') {
  const selectedMethodType = _.get(json, 'grpc.methodType');
  selectedMethodType && (transformedJson.request.methodType = selectedMethodType);
  const protoPath = _.get(json, 'grpc.protoPath');
  protoPath && (transformedJson.request.protoPath = protoPath);
  transformedJson.request.auth.mode = _.get(json, 'grpc.auth', 'none');
  transformedJson.request.body = _.get(json, 'body', {
    mode: 'grpc',
    grpc: _.get(json, 'body.grpc', [{ name: 'message 1', content: '{}' }])
  });
}
```

**WebSocket 请求**：

```javascript
else if (requestType === 'ws-request') {
  transformedJson.request.auth.mode = _.get(json, 'ws.auth', 'none');
  transformedJson.request.body = _.get(json, 'body', {
    mode: 'ws',
    ws: _.get(json, 'body.ws', [{ name: 'message 1', content: '{}' }])
  });
}
```

**HTTP / GraphQL 请求**：

```javascript
else {
  transformedJson.request.params = _.get(json, 'params', []);
  transformedJson.request.auth.mode = _.get(json, 'http.auth', 'none');
  transformedJson.request.body.mode = _.get(json, 'http.body', 'none');
}
```

#### 4. OAuth2 额外参数处理

```javascript
const hasOauth2GrantType = json?.auth?.oauth2?.grantType;
if (hasOauth2GrantType) {
  const additionalParameters = getOauth2AdditionalParameters(json);
  const hasAdditionalParameters = Object.keys(additionalParameters || {}).length > 0;
  if (hasAdditionalParameters) {
    transformedJson.request.auth.oauth2.additionalParameters = additionalParameters;
  }
}
```

### 转换后的内部对象结构

```javascript
{
  type: 'http-request',
  name: 'Send Bulk SMS',
  seq: 1,
  settings: { encodeUrl: true, timeout: 0 },
  tags: ['foo', 'bar'],
  request: {
    method: 'GET',
    url: 'https://api.textlocal.in/send/:id',
    headers: [{ name, value, enabled, uid, ... }],
    params: [{ name, value, type, enabled, uid, ... }],
    auth: {
      mode: 'bearer',
      bearer: { token: '123' },
      oauth2: { grantType: 'authorization_code', additionalParameters: { ... }, ... }
    },
    body: {
      mode: 'json',
      json: '{ "hello": "world" }'
    },
    script: { req: '...', res: '...' },
    vars: { req: [...], res: [...] },
    assertions: [...],
    tests: '...',
    docs: '...'
  },
  examples: [...]
}
```

---

## 第三层：Schema 验证

### 核心实现文件

- `packages/bruno-schema/src/collections/index.js`

### 使用技术

- **Yup**：JavaScript Schema 构建和验证库

### 核心 Schema 结构

#### 1. Item Schema

```javascript
const itemSchema = Yup.object({
  uid: uidSchema,  // 21位字母数字唯一ID
  type: Yup.string().oneOf([
    'http-request', 'graphql-request', 'folder', 'js', 'grpc-request', 'ws-request'
  ]).required(),
  seq: Yup.number().min(1),
  name: Yup.string().min(1).required(),
  tags: Yup.array().of(Yup.string()),
  request: Yup.mixed(),  // 根据 type 条件验证
  settings: Yup.mixed(), // 根据 type 条件验证
  fileContent: Yup.string().when('type', { is: 'js', then: Yup.string() }),
  root: Yup.mixed().when('type', { is: 'folder', then: folderRootSchema }),
  items: Yup.lazy(() => Yup.array().of(itemSchema)),  // 递归嵌套
  examples: Yup.array().of(exampleSchema),
  filename: Yup.string().nullable(),
  pathname: Yup.string().nullable()
})
```

#### 2. Request Schema

```javascript
const requestSchema = Yup.object({
  url: Yup.string().defined(),
  method: Yup.string().min(1).required(),
  headers: Yup.array().of(keyValueSchema).required(),
  params: Yup.array().of(requestParamsSchema).required(),
  auth: authSchema,  // 包含各种认证方式
  body: requestBodySchema,
  script: { req: Yup.string().nullable(), res: Yup.string().nullable() },
  vars: { req: Yup.array().of(varsSchema), res: Yup.array().of(varsSchema) },
  assertions: Yup.array().of(assertionSchema).nullable(),
  tests: Yup.string().nullable(),
  docs: Yup.string().nullable()
})
```

#### 3. Collection Schema

```javascript
const collectionSchema = Yup.object({
  version: Yup.string().oneOf(['1']).required(),
  uid: uidSchema,
  name: Yup.string().min(1).required(),
  items: Yup.array().of(itemSchema),  // 递归包含所有 items/folders
  activeEnvironmentUid: Yup.string().length(21).nullable(),
  environments: Yup.array().of(environmentSchema),
  pathname: Yup.string().nullable(),
  runnerResult: Yup.object(),
  runtimeVariables: Yup.object(),
  workspaceProcessEnvVariables: Yup.object().default({}),
  brunoConfig: Yup.object(),
  root: folderRootSchema  // 集合级别的继承配置
})
```

---

---

## 第四层：文件系统遍历与集合构建

### 核心实现文件

- `packages/bruno-cli/src/utils/collection.js` (CLI 端集合构建)
- `packages/bruno-electron/src/utils/collection.js` (Electron 端辅助函数)
- `packages/bruno-electron/src/ipc/collection.js` (Electron IPC 处理)

### 1. 目录递归遍历与文件发现

#### 文件类型识别

```
collectionRoot/
├── bruno.json         # 集合配置文件（版本、名称等）
├── collection.bru      # 集合根继承配置（全局 headers/vars/script/auth）
├── environments/        # 环境变量目录
│   └── dev.bru
├── folder1/
│   ├── folder.bru      # 文件夹级继承配置
│   ├── request1.bru    # 请求文件
│   └── subfolder/
│       ├── folder.bru
│       └── request2.bru
└── request3.bru
```

#### 遍历策略 (`createCollectionJsonFromPathname`)

**核心遍历逻辑（bruno-cli/src/utils/collection.js）：

```javascript
const traverse = (currentPath) => {
  if (currentPath.includes('node_modules')) return [];
  const currentDirItems = [];

  // 1. 读取目录所有文件
  const files = fs.readdirSync(currentPath);

  for (const file of files) {
    const filePath = path.join(currentPath, file);
    const stats = fs.lstatSync(filePath);

    if (stats.isDirectory()) {
      // 2. 跳过特殊目录
      if (filePath === environmentsPath || file === '.git' || file === 'node_modules') continue;
      
      // 3. 递归处理子目录（深度优先）
      const folderItem = { 
        name: file, 
        pathname: filePath, 
        type: 'folder', 
        items: traverse(filePath) 
      };
      
      // 4. 读取 folder.bru（如果存在）
      const folderRoot = getFolderRoot(filePath, format);
      if (folderRoot) {
        folderItem.root = folderRoot;
        folderItem.seq = folderRoot.meta?.seq;  // 文件夹排序序号
      }
      currentDirItems.push(folderItem);
    } else {
      // 5. 跳过 collection.bru 和 folder.bru，只处理请求文件 (*.bru)
      if (file === collectionFile || file === folderFile || path.extname(filePath) !== ext) continue;
      
      try {
        // 6. 解析请求文件
        const requestItem = parseRequest(fs.readFileSync(filePath, 'utf8'), { format });
        currentDirItems.push({ name: file, ...requestItem, pathname: filePath });
      } catch (err) {
        // 7. 异常处理：记录警告，跳过当前文件，继续处理其他文件
        console.warn(chalk.yellow(`Warning: Skipping invalid file ${filePath}\nError: ${err.message}`));
        global.brunoSkippedFiles = global.brunoSkippedFiles || [];
        global.brunoSkippedFiles.push({ path: filePath, error: err.message });
      }
    }
  }

  // 8. 排序：文件夹在前，请求在后
  const folders = sortByNameThenSequence(currentDirItems.filter((i) => i.type === 'folder'));
  const requests = currentDirItems.filter((i) => i.type !== 'folder').sort((a, b) => a.seq - b.seq);
  return folders.concat(requests);
};
```

### 2. 异常文件跳过策略 (CLI 模式)

#### 单文件解析失败不影响整体加载

```javascript
// 位于 bruno-cli/src/utils/collection.js
try {
  const requestItem = parseRequest(fs.readFileSync(filePath, 'utf8'), { format });
  currentDirItems.push({ name: file, ...requestItem, pathname: filePath });
} catch (err) {
  // 解析失败：发出警告，跳过当前文件，继续处理其他文件
  console.warn(chalk.yellow(`Warning: Skipping invalid file ${filePath}\nError: ${err.message}`));
  // 记录跳过的文件（供后续参考）
  global.brunoSkippedFiles = global.brunoSkippedFiles || [];
  global.brunoSkippedFiles.push({ path: filePath, error: err.message });
}
```

#### Meta 块快速解析（Electron 端容错路径）

当完整解析失败时，使用轻量级正则表达式仅提取 meta 块用于侧边栏显示：

```javascript
const parseBruFileMeta = (data) => {
  try {
    const metaRegex = /meta\s*{\s*([\s\S]*?)\s*}/;
    const match = data?.match?.(metaRegex);
    if (match) {
      const metaContent = match[1].trim();
      const lines = metaContent.replace(/\r\n/g, '\n').split('\n');
      const metaJson = {};
      lines.forEach((line) => {
        const [key, value] = line.split(':').map((str) => str.trim());
        if (key && value) { metaJson[key] = isNaN(value) ? value : Number(value); }
      });

      // 转换为应用可识别的最小结构
      let requestType = metaJson.type;
      requestType = requestType === 'http' ? 'http-request' 
        : requestType === 'graphql' ? 'graphql-request' 
        : 'http-request';
      const sequence = metaJson.seq;
      const transformedJson = {
        type: requestType,
        name: metaJson.name,
        seq: !isNaN(sequence) ? Number(sequence) : 1,
        settings: {},
        tags: metaJson.tags || [],
        request: { method: '', url: '', params: [], headers: [], auth: { mode: 'none' }, body: { mode: 'none' },
          script: {}, vars: {}, assertions: [], tests: '', docs: '' }
      };
      return transformedJson;  // 可用于显示树节点名称
    }
  } catch (err) {
    console.error('Error reading file:', err);
    return null;
  }
};
```

### 3. collection.bru 与 folder.bru 的作用

#### 特殊文件结构

| 文件 | 位置 | 作用 |
|------|------|------|
| `collection.bru` | 集合根目录 | 定义全局继承配置（Headers/Vars/Script/Auth/Tests） |
| `folder.bru` | 子文件夹 | 定义文件夹级继承配置，覆盖父级配置 |

#### collection.bru 内容结构示例

```
# collection.bru 不需要 meta.name（名称从 bruno.json 读取
auth {
  mode: bearer
  token: default_token
}

headers {
  X-App-Name: bruno
}

script:pre-request {
  console.log("collection pre-request script");
}

vars:pre-request {
  baseUrl: https://api.example.com
}

tests {
  console.log("collection test script");
}
```

#### folder.bru 内容结构示例

```
meta {
  name: folder_name  # folder.bru 需要 name 用于显示
  seq: 1             # 文件夹排序序号（影响同级文件夹排序）
}

auth {
  mode: bearer
  token: folder_token
}

headers {
  X-Folder-Header: value
}
```

### 4. 目录递归建树

#### 构建嵌套对象树

```
文件系统路径:
  collection/
    bruno.json
    collection.bru
    folderA/
      folder.bru
      req1.bru
      subfolder/
        folder.bru
        req2.bru

转换为内部结构:
{
  brunoConfig: { name: "collection", version: "1", ... },
  format: "bru",
  root: { request: { headers, vars, script, auth, tests } },  // collection.bru 解析结果
  pathname: "/path/to/collection",
  items: [
    {
      type: "folder",
      name: "folderA",
      pathname: "/path/to/collection/folderA",
      root: { request: { ...folderA 级配置 } },
      seq: 1,
      items: [
        { type: "http-request", name: "req1", seq: 1, pathname: "...", request: {...} },
        {
          type: "folder",
          name: "subfolder",
          pathname: "...",
          root: { request: { ...subfolder 级配置 } },
          items: [ { type: "http-request", name: "req2", ... } ]
        }
      ]
    }
  ]
}
```

#### 关键辅助函数

```javascript
// 扁平化所有 items（便于查找）
const flattenItems = (items = []) => {
  const flattenedItems = [];
  const flatten = (itms, flattened) => {
    each(itms, (i) => {
      flattened.push(i);
      if (i.items && i.items.length) flatten(i.items, flattened);
    });
  };
  flatten(items, flattenedItems);
  return flattenedItems;
};

// 获取从 collection 到 item 的路径（用于继承合并）
const getTreePathFromCollectionToItem = (collection, _item) => {
  let path = [];
  let item = findItemInCollection(collection, _item.uid);
  while (item) {
    path.unshift(item);  // 从 request 到 collection 倒序插入
    item = findParentItemInCollection(collection, item.uid);
  }
  return path;  // [collectionRoot, folderA, subfolder, request]
};
```

### 5. seq 与名称排序算法

#### `sortByNameThenSequence` 核心逻辑

```javascript
const sortByNameThenSequence = (items) => {
  const isSeqValid = (seq) => Number.isFinite(seq) && Number.isInteger(seq) && seq > 0;
  
  // Step 1: 所有 items 先按名称字母序排序
  const alphabeticallySorted = [...items].sort((a, b) => 
    a.name && b.name && a.name.localeCompare(b.name));
  
  // Step 2: 分离出带/不带有效 seq 的 items
  const withoutSeq = alphabeticallySorted.filter((f) => !isSeqValid(f['seq']));
  const withSeq = alphabeticallySorted
    .filter((f) => isSeqValid(f['seq']))
    .sort((a, b) => a.seq - b.seq);  // 带 seq 的先按 seq 排序
  
  // Step 3: 将带 seq 的 items 插入到指定位置 (seq - 1)
  withSeq.forEach((item) => {
    const position = item.seq - 1;
    const existingItem = withoutSeq[position];
    
    // seq 冲突处理：相同 seq 的 items 合并在一起，保持字母序
    const hasItemWithSameSeq = Array.isArray(existingItem)
      ? existingItem?.[0]?.seq === item.seq
      : existingItem?.seq === item.seq;
    
    if (hasItemWithSameSeq) {
      const newGroup = Array.isArray(existingItem)
        ? [...existingItem, item]
        : [existingItem, item];
      withoutSeq.splice(position, 1, newGroup);
    } else {
      withoutSeq.splice(position, 0, item);
    }
  });
  
  return withoutSeq.flat();  // 展平嵌套数组
};
```

#### 集合整体排序策略

```javascript
const sortCollection = (collection) => {
  const items = collection.items || [];
  // Step 1: 分离 folders 与 requests
  let folderItems = filter(items, (item) => item.type === 'folder');
  let requestItems = filter(items, (item) => item.type !== 'folder');
  
  // Step 2: 文件夹 按 seq+name 排序
  folderItems = sortByNameThenSequence(folderItems);
  
  // Step 3: 请求 仅按 seq 排序（请求无名称排序需求）
  requestItems = requestItems.sort((a, b) => a.seq - b.seq);
  
  // Step 4: 文件夹始终排在请求前面
  collection.items = folderItems.concat(requestItems);
  
  // Step 5: 递归处理所有子文件夹
  each(folderItems, (item) => sortCollection(item));
};
```

### 6. root / request 继承合并机制

#### 继承优先级（从低到高）

```
collection.bru (root)
    ↓
  folderA/folder.bru (root)
    ↓
  folderA/subfolder/folder.bru (root)
    ↓
  request.bru (request) —— 最高优先级，可覆盖上层
```

#### (1) Headers 合并

```javascript
const mergeHeaders = (collection, request, requestTreePath, options = {}) => {
  const { includeDisabledHeaders = false } = options;
  let headers = new Map();        // 启用的 headers（后入覆盖先入）
  let disabledHeaders = new Map();  // 禁用的 headers
  
  // 1. collection 级别 headers（最先加入）
  const collectionRoot = collection?.draft?.root || collection?.root || {};
  let collectionHeaders = get(collectionRoot, 'request.headers', []);
  collectionHeaders.forEach((header) => {
    if (header.enabled) {
      headers.set(header.name.toLowerCase(), header.value);
    } else if (header.name?.length > 0) {
      disabledHeaders.set(header.name, header.value);
    }
  });
  
  // 2. 遍历 folder 路径（从上到下，下层覆盖上层）
  for (let i of requestTreePath) {
    if (i.type === 'folder') {
      const folderRoot = i?.draft || i?.root;
      let folderHeaders = get(folderRoot, 'request.headers', []);
      folderHeaders.forEach((header) => {
        if (header.enabled) {
          headers.set(header.name.toLowerCase(), header.value);
        } else if (header.name?.length > 0) {
          disabledHeaders.set(header.name, header.value);
        }
      });
    }
  }
  
  // 3. request 级别 headers（最后加入，覆盖所有上层）
  for (let i of requestTreePath) {
    if (i.type !== 'folder') {
      const requestHeaders = i?.draft ? get(i, 'draft.request.headers', []) : get(i, 'request.headers', []);
      requestHeaders.forEach((header) => {
        if (header.enabled) {
          headers.set(header.name.toLowerCase(), header.value);
        } else if (header.name?.length > 0) {
          disabledHeaders.set(header.name, header.value);
        }
      });
    }
  }
  
  // 返回最终合并结果
  request.headers = [
    ...Array.from(headers, ([name, value]) => ({ name, value, enabled: true })),
    ...(includeDisabledHeaders ? Array.from(disabledHeaders, ([name, value]) => ({ name, value, enabled: false })) : [])
  ];
};
```

#### (2) Vars 合并

```javascript
const mergeVars = (collection, request, requestTreePath = []) => {
  let reqVars = new Map();  // pre-request variables
  
  // collection 级别
  const collectionRoot = collection?.draft?.root || collection?.root || {};
  let collectionRequestVars = get(collectionRoot, 'request.vars.req', []);
  let collectionVariables = {};
  collectionRequestVars.forEach((_var) => {
    if (_var.enabled) {
      reqVars.set(_var.name, _var.value);
      collectionVariables[_var.name] = _var.value;
    }
  });
  
  // folder 级别（从上到下）
  let folderVariables = {};
  let requestVariables = {};
  for (let i of requestTreePath) {
    if (i.type === 'folder') {
      const folderRoot = i?.draft || i?.root;
      let vars = get(folderRoot, 'request.vars.req', []);
      vars.forEach((_var) => {
        if (_var.enabled) {
          reqVars.set(_var.name, _var.value);
          folderVariables[_var.name] = _var.value;
        }
      });
    }
  }
  
  // request 级别
  for (let i of requestTreePath) {
    if (i.type !== 'folder') {
      const vars = i?.draft ? get(i, 'draft.request.vars.req', []) : get(i, 'request.vars.req', []);
      vars.forEach((_var) => {
        if (_var.enabled) {
          reqVars.set(_var.name, _var.value);
          requestVariables[_var.name] = _var.value;
        }
      });
    }
  }
  
  // 存储层级信息（用于调试）
  request.collectionVariables = collectionVariables;
  request.folderVariables = folderVariables;
  request.requestVariables = requestVariables;
  
  // 最终合并结果
  if (request?.vars) {
    request.vars.req = Array.from(reqVars, ([name, value]) => ({
      name, value, enabled: true, type: 'request'
    }));
  }
  
  // post-response vars 同理（略）
  let resVars = new Map();
  // ... 处理 res vars
};
```

#### (3) Auth 合并（最近有效原则）

```javascript
const mergeAuth = (collection, request, requestTreePath) => {
  // Step 1: 从 collection 级别开始，作为默认
  const collectionRoot = collection?.draft?.root || collection?.root || {};
  let collectionAuth = get(collectionRoot, 'request.auth', { mode: 'none' });
  let effectiveAuth = collectionAuth;
  let lastFolderWithAuth = null;
  
  // Step 2: 遍历路径，找到最近的非 inherit / non none 的 auth
  for (let i of requestTreePath) {
    if (i.type === 'folder') {
      const folderRoot = i?.draft || i?.root;
      const folderAuth = get(folderRoot, 'request.auth');
      // 只有 mode 有效且非 inherit / none 时才覆盖
      if (folderAuth && folderAuth.mode && 
          folderAuth.mode !== 'none' && folderAuth.mode !== 'inherit') {
        effectiveAuth = folderAuth;
        lastFolderWithAuth = i;
      }
    }
  }
  
  // Step 3: request 级别为 inherit 时，使用上层计算结果
  if (request.auth.mode === 'inherit') {
    request.auth = effectiveAuth;
    // OAuth2 特殊处理：标记凭证来源 folderUid
    if (effectiveAuth.mode === 'oauth2') {
      request.oauth2Credentials = {
        folderUid: lastFolderWithAuth?.uid || null,
        itemUid: null,
        mode: request.auth.mode
      };
    }
  }
};
```

#### (4) Script 与 Tests 合并

```javascript
const mergeScripts = (collection, request, requestTreePath, scriptFlow) => {
  const collectionRoot = collection?.draft?.root || collection?.root || {};
  let collectionPreReqScript = get(collectionRoot, 'request.script.req', '');
  let collectionPostResScript = get(collectionRoot, 'request.script.res', '');
  let collectionTests = get(collectionRoot, 'request.tests', '');
  
  // 收集所有 folder 级别的 script
  let combinedPreReqScript = [];
  let combinedPostResScript = [];
  let combinedTests = [];
  for (let i of requestTreePath) {
    if (i.type === 'folder') {
      const folderRoot = i?.draft || i?.root;
      let preReqScript = get(folderRoot, 'request.script.req', '');
      let postResScript = get(folderRoot, 'request.script.res', '');
      let tests = get(folderRoot, 'request.tests', '');
      if (preReqScript?.trim()) combinedPreReqScript.push(preReqScript);
      if (postResScript?.trim()) combinedPostResScript.push(postResScript);
      if (tests?.trim()) combinedTests.push(tests);
    }
  }
  
  // 原始 request 级别 script（用于保留元信息）
  const originalPreReqScript = request?.script?.req || '';
  const originalPostResScript = request?.script?.res || '';
  const originalTests = request?.tests || '';
  
  // pre-request: sequential 模式 → 从 collection 到 folder 到 request（顺序执行）
  // pre-request: non-sequential 模式 → 相同顺序
  const preReqScripts = [
    collectionPreReqScript,
    ...combinedPreReqScript,
    originalPreReqScript
  ];
  
  // post-response: sequential 模式 → collection → folder → request（自上而下）
  // post-response: non-sequential 模式 → request → folder → collection（自下而上，反转）
  // tests: 同 post-response 顺序
  if (scriptFlow === 'sequential') {
    const postResScripts = [collectionPostResScript, ...combinedPostResScript, originalPostResScript];
    const testScripts = [collectionTests, ...combinedTests, originalTests];
    // ... 合并
  } else {
    const postResScripts = [originalPostResScript, ...[...combinedPostResScript].reverse(), collectionPostResScript];
    const testScripts = [originalTests, ...[...combinedTests].reverse(), collectionTests];
    // ... 合并
  }
  
  // 每个 script 包裹在 async IIFE 中，隔离作用域，避免变量冲突
  const wrapScriptInClosure = (script) => {
    if (!script?.trim()) return '';
    return `await (async () => {\n${script}\n})();`;
  };
  
  // 记录元信息（栈跟踪映射）
  request.script.reqMetadata = {
    requestStartLine: 9,
    requestEndLine: 11,
    segments: [
      { startLine: 1, endLine: 3, source: 'collection', fileName: 'collection.bru' },
      { startLine: 5, endLine: 7, source: 'folder', fileName: 'folderA/folder.bru' }
    ],
    requestScriptContent: originalPreReqScript  // 原始内容（便于比对差异）
  };
};
```

### 7. 完整集合构建流程图

```
文件系统目录
     │
     ▼
深度优先遍历 (traverse)
     │
     ▼
┌─────────────────────────────────────────────┐
│   文件分类处理                                │
│   ├── bruno.json → 集合配置                 │
│   ├── collection.bru → 全局 root 配置        │
│   ├── folder.bru → 文件夹 root 配置          │
│   └── *.bru (≠ folder.bru) → 请求解析        │
│                                              │
│   解析失败降级策略：                          │
│   CLI: 警告 + 跳过； Electron: meta 快解析   │
└─────────────────────┬───────────────────────┘
                      │
                      ▼
            构建嵌套 items 树
                      │
                      ▼
┌─────────────────────────────────────────────┐
│   排序阶段                                    │
│   ├── folders: sortByNameThenSequence()      │
│   │   ├── 先按名称字母序                      │
│   │   ├── 有效 seq 的按 seq 插入位置         │
│   │   └── seq 冲突时合并保持字母序           │
│   ├── requests: 仅按 seq 数字排序            │
│   └── folders 始终排在 requests 前面          │
└─────────────────────┬───────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────┐
│   继承合并阶段 (Runner 执行前)                │
│   path = [collectionRoot, folderA, request]  │
│                                              │
│   Headers:   Map 去重，后入覆盖先入           │
│   Vars:      Map 去重，后入覆盖先入           │
│   Auth:      最近有效原则（跳过 inherit/none）│
│   Script:    顺序拼接 + IIFE 隔离             │
│   Tests:     sequential/反转 两种策略        │
└─────────────────────┬───────────────────────┘
                      │
                      ▼
            Collection 内存对象
```

---

## 完整解析流程图

```
           .bru 文件 (文本)
                │
                ▼
    ┌───────────────────────────┐
    │   bruno-lang/bruToJsonV2  │
    │   - Ohm Grammar 解析       │
    │   - Semantics AST 转换     │
    │   - 块类型映射处理         │
    └─────────────┬─────────────┘
                  │
                  ▼
          原始 JSON 对象
    { meta, http, headers, params, 
      auth, body, vars, assertions... }
                  │
                  ▼
    ┌───────────────────────────┐
    │ bruno-filestore/parseBru  │
    │   - 请求类型映射           │
    │   - 字段规范化重命名       │
    │   - OAuth2 额外参数合并    │
    │   - Example 嵌套解析       │
    └─────────────┬─────────────┘
                  │
                  ▼
          内部 Item 对象
    { uid, type, name, seq, 
      request { method, url, headers, 
                params, auth, body, ... },
      settings, tags, examples }
                  │
                  ▼
    ┌───────────────────────────┐
    │   bruno-schema 验证        │
    │   - Yup Schema 验证        │
    │   - 字段类型检查           │
    │   - 嵌套结构验证           │
    └─────────────┬─────────────┘
                  │
                  ▼
          Collection 对象
    { version, uid, name, 
      items: [递归嵌套],
      environments, root, ... }
```

---

## 关键数据结构转换表

| 原始 bru 字段 | parseBruRequest 后 | 最终 Schema 字段 |
|--------------|-------------------|-----------------|
| `meta.name` | `name` | `name` |
| `meta.type` | `type` (http → http-request) | `type` |
| `meta.seq` | `seq` | `seq` |
| `meta.tags` | `tags` | `tags` |
| `http.method` | `request.method` | `request.method` |
| `http.url` | `request.url` | `request.url` |
| `http.auth` | `request.auth.mode` | `request.auth.mode` |
| `http.body` | `request.body.mode` | `request.body.mode` |
| `headers` | `request.headers` | `request.headers` |
| `params` | `request.params` | `request.params` |
| `auth.*` | `request.auth.*` | `request.auth.*` |
| `body.*` | `request.body.*` | `request.body.*` |
| `script.req` | `request.script.req` | `request.script.req` |
| `script.res` | `request.script.res` | `request.script.res` |
| `vars.req` | `request.vars.req` | `request.vars.req` |
| `vars.res` | `request.vars.res` | `request.vars.res` |
| `assertions` | `request.assertions` | `request.assertions` |
| `tests` | `request.tests` | `request.tests` |
| `docs` | `request.docs` | `request.docs` |
| `settings` | `settings` | `settings` |
| `examples` | `examples` | `examples` |

---

## 特殊处理要点

### 1. 禁用标记 (~)

在 bru 文件中，前缀 `~` 表示该条目被禁用：

```
headers {
  ~transaction-id: {{transactionId}}  // enabled: false
}
```

处理位置：`mapPairListToKeyValPairs` 函数

### 2. 注解 (Annotations)

支持 `@name(value)` 形式的注解，用于元数据标记：

```
params:query {
  @sensitive(password)
  secret: mypassword
}
```

处理位置：`pair` 语义处理函数，存储在 Symbol 键 `ANNOTATIONS_KEY`

### 3. 多行文本缩进

文本块内容自动缩进处理：

```javascript
// 移除每行前导空格
multilinetextblock(_1, content, _2, _3, contentType) {
  const multilineString = content.sourceString
    .split('\n')
    .map((line) => line.slice(4))  // 移除 4 空格缩进
    .join('\n');
  // ...
}
```

### 4. OAuth2 Grant Type 条件字段

不同 grant_type 有不同字段，使用 Yup `.when()` 条件验证：

```javascript
clientId: Yup.string().when('grantType', {
  is: (val) => ['client_credentials', 'password', 'authorization_code', 'implicit'].includes(val),
  then: Yup.string().nullable(),
  otherwise: Yup.string().nullable().strip()
})
```

### 5. 递归嵌套集合结构

使用 `Yup.lazy()` 实现 folder 的递归嵌套：

```javascript
items: Yup.lazy(() => Yup.array().of(itemSchema))
```

---

## 文件系统集成入口

### 核心文件

- `packages/bruno-filestore/src/index.ts`

### 对外 API

```javascript
// 请求解析
export const parseRequest = (content: string, options: ParseOptions): any => {
  if (options.format === 'bru') return parseBruRequest(content);
  // ...
};

// 集合解析
export const parseCollection = (content: string, options: ParseOptions): any => {
  if (options.format === 'bru') return parseBruCollection(content);
  // ...
};

// 环境解析
export const parseEnvironment = (content: string, options: ParseOptions): any => {
  if (options.format === 'bru') return parseBruEnvironment(content);
  // ...
};

// Worker 异步解析（性能优化）
export const parseRequestViaWorker = async (content: string, options): Promise<any> => {
  const fileParserWorker = getWorkerInstance();
  return await fileParserWorker.parseRequest(content, options.format);
};
```

---

## 总结

整个 bru 文件解析过程体现了清晰的分层架构设计：

1. **语法层** - 使用声明式语法解析器，关注点分离，易于扩展新的块类型
2. **转换层** - 标准化数据结构，处理不同请求类型的特殊逻辑
3. **验证层** - Schema 驱动的验证，保证数据完整性和类型安全

关键设计模式：
- **语义动作** (Semantic Actions)：语法解析与业务逻辑分离
- **条件 Schema**：根据请求类型动态验证字段
- **递归嵌套**：支持无限层级的文件夹结构
- **Worker 隔离**：CPU 密集型解析操作移至 Web Worker
