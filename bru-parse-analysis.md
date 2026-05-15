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
