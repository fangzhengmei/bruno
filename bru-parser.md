# Bru 文件解析机制详解

## 1. 概述

Bru 是 Bruno API 客户端中用于存储 HTTP 请求定义的纯文本文件格式。解析器负责将 Bru 纯文本文件转换为可执行的 JSON 结构，支持发送 HTTP 请求、认证、脚本执行等操作。

## 2. Bru 文件语法

Bru 文件经历了两个主要版本：V1 和 V2。

### 2.1 V1 格式（旧版）

```
name Send Bulk SMS
method GET
url https://api.textlocal.in/send/
body-mode json
seq 1

params
  1 apiKey secret
  0 numbers 998877665
/params

headers
  1 content-type application/json
/headers

body(type=json)
  {
    "key": "value"
  }
/body

script
  console.log('hello');
/script

tests
  expect(response.status).to.equal(200);
/tests
```

### 2.2 V2 格式（新版，推荐）

```
meta {
  name: Send Bulk SMS
  type: http
  seq: 1
  tags: [
    foo
    bar
  ]
}

get {
  url: https://api.textlocal.in/send/:id
  body: json
  auth: bearer
}

params:query {
  apiKey: secret
  numbers: 998877665
  "key with spaces": is allowed
  ~disabled-key: value
}

params:path {
  id: 123
}

headers {
  content-type: application/json
  Authorization: Bearer 123
}

body:json {
  {
    "hello": "world"
  }
}
```

#### 2.2.1 语法特性

| 特性 | 说明 |
|------|------|
| **块结构** | 使用 `tagname { ... }` 格式定义块 |
| **键值对** | `key: value` 格式，支持带引号的键名 |
| **禁用项** | 键名前加 `~` 表示禁用 |
| **列表** | `[ ... ]` 格式，用于 tags 等 |
| **多行文本** | 支持 JSON、XML、文本、GraphQL 等多种 body 格式 |
| **注释** | 暂不支持 |

#### 2.2.2 支持的块类型

```
# 元数据
meta { ... }

# HTTP 方法
get { ... }
post { ... }
put { ... }
delete { ... }
patch { ... }
options { ... }
head { ... }
connect { ... }
trace { ... }
http { ... }  # 自定义方法

# gRPC 和 WebSocket
grpc { ... }
ws { ... }

# 参数
params:query { ... }
params:path { ... }
query { ... }  # 别名

# 头信息
headers { ... }
metadata { ... }  # gRPC 元数据

# 认证
auth:basic { ... }
auth:bearer { ... }
auth:digest { ... }
auth:oauth1 { ... }
auth:oauth2 { ... }
auth:awsv4 { ... }
auth:wsse { ... }
auth:apikey { ... }
auth:ntlm { ... }

# 请求体
body:json { ... }
body:text { ... }
body:xml { ... }
body:sparql { ... }
body:graphql { ... }
body:graphql:vars { ... }
body:form-urlencoded { ... }
body:multipart-form { ... }
body:file { ... }

# 变量和断言
vars:pre-request { ... }
vars:post-response { ... }
assert { ... }

# 脚本和测试
script:pre-request { ... }
script:post-response { ... }
tests { ... }

# 文档
docs { ... }

# 示例
example { ... }

# 设置
settings { ... }
```

## 3. 解析器架构

### 3.1 技术栈

| 组件 | 说明 |
|------|------|
| **Ohm.js** | V2 解析器使用的 PEG（Parsing Expression Grammar）语法解析库 |
| **arcsecond** | V1 解析器使用的函数式解析器组合子库 |
| **Lodash** | 用于对象合并和数据转换 |

### 3.2 核心文件结构

```
packages/bruno-lang/
├── src/
│   └── index.js          # 入口文件，导出所有版本
├── v1/
│   ├── src/
│   │   ├── index.js          # V1 主解析器
│   │   ├── inline-tag.js     # 行内标签解析
│   │   ├── params-tag.js     # 参数标签解析
│   │   ├── headers-tag.js    # 头信息标签解析
│   │   ├── body-tag.js       # 请求体标签解析
│   │   ├── script-tag.js     # 脚本标签解析
│   │   ├── tests-tag.js      # 测试标签解析
│   │   ├── env-vars-tag.js   # 环境变量标签解析
│   │   └── utils.js          # 工具函数
│   └── tests/
└── v2/
    ├── src/
    │   ├── bruToJson.js      # V2 主解析器（Ohm.js 语法定义）
    │   ├── jsonToBru.js      # JSON 转 Bru
    │   ├── envToJson.js      # 环境文件解析
    │   ├── jsonToEnv.js      # JSON 转环境文件
    │   ├── dotenvToJson.js   # .env 文件解析
    │   ├── collectionBruToJson.js  # 集合文件解析
    │   ├── jsonToCollectionBru.js  # JSON 转集合文件
    │   ├── utils.js          # 工具函数
    │   └── example/          # 示例解析
    └── tests/
```

## 4. 解析步骤（V2 版本）

### 4.1 语法定义（Ohm Grammar）

解析器首先使用 Ohm.js 定义完整的 Bru 语法：

```javascript
const grammar = ohm.grammar(`Bru {
  BruFile = (meta | http | grpc | ws | query | params | headers | metadata | auths | bodies | varsandassert | script | tests | settings | docs | example)*
  
  # 字典块 - 键值对形式
  dictionary = st* "{" st* pairlist? tagend
  pairlist = optionalnl* pair (~tagend stnl* pair)* (~tagend space)*
  pair = st* pairannotations st* (quoted_key | key) st* ":" st* value st*
  
  # 文本块 - 自由文本形式
  textblock = textline (~tagend nl textline)*
  
  # 列表块 - 列表项形式
  list = st* "[" nl+ listitems? st* nl+ st* "]"
  
  # ... 更多语法规则
}`);
```

### 4.2 语义操作（Semantic Actions）

语法匹配成功后，通过语义属性（Semantics）将 AST 转换为 JSON：

```javascript
const sem = grammar.createSemantics().addAttribute('ast', {
  BruFile(tags) {
    // 合并所有标签的解析结果
    return _.reduce(tags.ast, (result, item) => {
      return _.mergeWith(result, item, concatArrays);
    }, {});
  },
  
  // 字典块处理
  dictionary(_1, _2, _3, pairlist, _4) {
    return pairlist.ast;
  },
  
  // 键值对处理
  pair(_1, annotations, _2, key, _3, _4, _5, value, _6) {
    let res = {};
    res[key.ast] = value.ast ? value.ast.trim() : '';
    // 处理注解
    const annotationList = annotations.ast;
    if (annotationList && annotationList.length > 0) {
      res[ANNOTATIONS_KEY] = annotationList;
    }
    return res;
  },
  
  // HTTP 方法处理
  get(_1, dictionary) {
    return {
      http: {
        method: 'get',
        ...mapPairListToKeyValPair(dictionary.ast)
      }
    };
  },
  
  // Headers 处理
  headers(_1, dictionary) {
    return {
      headers: mapPairListToKeyValPairs(dictionary.ast)
    };
  },
  
  // Body JSON 处理
  bodyjson(_1, _2, _3, _4, textblock, _5) {
    return {
      body: {
        json: outdentString(textblock.sourceString)
      }
    };
  }
});
```

### 4.3 解析流程

```
Bru 纯文本
    ↓
[Ohm Grammar 匹配]
    ↓
语法验证
    ↓
[Semantic Actions 转换]
    ↓
中间 AST（抽象语法树）
    ↓
[数据转换与合并]
    ↓
最终 JSON 结构（@usebruno/lang 输出）
    ↓
[Filestore 层归一化]
    ↓
可执行请求对象（http-request / graphql-request / grpc-request / ws-request）
```

### 4.4 关键转换函数

#### 4.4.1 `mapPairListToKeyValPairs

将字典块的键值对列表转换为标准格式：

```javascript
const mapPairListToKeyValPairs = (pairList = [], parseEnabled = true) => {
  return _.map(pairList[0], (pair) => {
    let name = _.keys(pair)[0];
    let value = pair[name];
    
    // 解析启用/禁用状态
    let enabled = true;
    if (name && name.length && name.charAt(0) === '~') {
      name = name.slice(1);
      enabled = false;
    }
    
    return { name, value, enabled };
  });
};
```

#### 4.4.2 `outdentString`

移除文本块的缩进：

```javascript
const outdentString = (str, spaces = 2) => {
  const spacesRegex = new RegExp(`^ {${spaces}}`);
  return str
    .split(/\r\n|\r|\n/)
    .map((line) => line.replace(spacesRegex, ''))
    .join('\n');
};
```

## 5. 生成的中间结构（JSON）

### 5.1 @usebruno/lang 输出的原始 JSON 结构

```json
{
  "meta": {
    "name": "Send Bulk SMS",
    "type": "http",
    "seq": "1",
    "tags": ["foo", "bar"]
  },
  "http": {
    "method": "get",
    "url": "https://api.textlocal.in/send/:id",
    "body": "json",
    "auth": "bearer"
  },
  "params": [
    {
      "name": "apiKey",
      "value": "secret",
      "type": "query",
      "enabled": true
    },
    {
      "name": "id",
      "value": "123",
      "type": "path",
      "enabled": true
    }
  ],
  "headers": [
    {
      "name": "content-type",
      "value": "application/json",
      "enabled": true
    }
  ],
  "auth": {
    "bearer": {
      "token": "123"
    },
    "basic": {
      "username": "john",
      "password": "secret"
    }
  },
  "body": {
    "json": "{\n  \"hello\": \"world\"\n}",
    "text": "This is a text body",
    "formUrlEncoded": [
      { "name": "apikey", "value": "secret", "enabled": true }
    ],
    "multipartForm": [
      { "name": "file", "type": "file", "value": ["path/to/file"], "enabled": true }
    ]
  },
  "vars": {
    "req": [
      { "name": "departingDate", "value": "2020-01-01", "local": false, "enabled": true }
    ],
    "res": [
      { "name": "token", "value": "$res.body.token", "local": false, "enabled": true }
    ]
  },
  "assertions": [
    { "name": "$res.status", "value": "200", "enabled": true }
  ],
  "script": {
    "req": "const foo = 'bar';"
  },
  "tests": "function onResponse(request, response) {\n  expect(response.status).to.equal(200);\n}",
  "docs": "This request needs auth token to be set in the headers."
}
```

### 5.2 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| meta | Object | 请求元数据（名称、类型、序号、标签） |
| http | Object | HTTP 请求信息（方法、URL 等） |
| params | Array | 请求参数数组（query/path） |
| headers | Array | 请求头数组 |
| auth | Object | 认证配置（支持多种认证方式） |
| body | Object | 请求体（支持多种格式） |
| vars | Object | 预请求和响应后变量 |
| assertions | Array | 断言列表 |
| script | Object | 预请求和响应后脚本 |
| tests | String | 测试脚本 |
| docs | String | 文档描述 |

## 6. Filestore 层归一化：从 Bru JSON 到可执行请求对象

### 6.1 类型映射表

| @usebruno/lang meta.type | Filestore 输出 type | Schema 类型 |
|---------------------------|-----------------------|-------------|
| http | http-request | HttpRequest |
| graphql | graphql-request | HttpRequest（复用） |
| grpc | grpc-request | GrpcRequest |
| ws | ws-request | WebSocketRequest |
| 空/其他 | http-request | HttpRequest |

### 6.2 核心转换函数：parseBruRequest

**文件位置**：`packages/bruno-filestore/src/formats/bru/index.ts:12-116`

#### 6.2.1 请求类型分支逻辑

```javascript
// 第一步：确定请求类型
let requestType = _.get(json, 'meta.type');
switch (requestType) {
  case 'http':
    requestType = 'http-request';
    break;
  case 'graphql':
    requestType = 'graphql-request';
    break;
  case 'grpc':
    requestType = 'grpc-request';
    break;
  case 'ws':
    requestType = 'ws-request';
    break;
  default:
    requestType = 'http-request';  // 默认回退到 HTTP
}
```

#### 6.2.2 URL 路径映射

| 请求类型 | 源字段路径 |
|----------|-----------|
| grpc-request | json.grpc.url |
| ws-request | json.ws.url |
| http-request | json.http.url |
| graphql-request | json.http.url |
| 默认 | json.http.url |

```javascript
const urlPath: Record<typeof requestType, string> = {
  'grpc-request': 'grpc.url',
  'ws-request': 'ws.url',
  'default': 'http.url'
};
```

### 6.3 Method 字段映射

| 请求类型 | 源字段 | 处理逻辑 | 默认值 |
|----------|--------|----------|--------|
| http-request | json.http.method | 转大写 | 空字符串 '' |
| graphql-request | json.http.method | 转大写 | 空字符串 '' |
| grpc-request | json.grpc.method | 不转大写，保留特殊字符 | 空字符串 '' |
| ws-request | json.http.method（隐式） | 走 HTTP 分支逻辑，转大写 | 空字符串 '' |

```javascript
// 所有请求类型都无条件设置 method 字段
method:
  requestType === 'grpc-request'
    ? _.get(json, 'grpc.method', '')  // gRPC 从 grpc.method 读，默认 ''
    : String(_.get(json, 'http.method') ?? '').toUpperCase()  // 其他类型（包括 ws）都走此分支
```

> **重要修正**：ws-request **包含** method 字段！由于所有请求类型共享同一 request 对象结构，method 字段在第 47-52 行被无条件设置。纯 WS 请求通常无 http 块，最终 method = 空字符串。

### 6.4 Headers / Metadata 字段映射

| 请求类型 | 源字段 | 目标字段 |
|----------|--------|----------|
| grpc-request | json.metadata | request.headers |
| 其他 | json.headers | request.headers |

```javascript
headers: requestType === 'grpc-request'
  ? _.get(json, 'metadata', [])
  : _.get(json, 'headers', [])
```

### 6.5 Auth 字段映射

| 请求类型 | auth.mode 源字段 |
|----------|------------------|
| grpc-request | json.grpc.auth |
| ws-request | json.ws.auth |
| http-request | json.http.auth |
| graphql-request | json.http.auth |
| 默认 | 'none' |

Auth 内容始终来自 `json.auth` 对象，包含所有认证方式的配置：

```javascript
auth: _.get(json, 'auth', {})

// 认证模式单独设置
if (requestType === 'grpc-request') {
  transformedJson.request.auth.mode = _.get(json, 'grpc.auth', 'none');
} else if (requestType === 'ws-request') {
  transformedJson.request.auth.mode = _.get(json, 'ws.auth', 'none');
} else {
  transformedJson.request.auth.mode = _.get(json, 'http.auth', 'none');
}
```

### 6.6 Body 字段映射（关键分支）

#### 6.6.1 Body Mode 映射

| 请求类型 | body.mode 来源 | 说明 |
|----------|---------------|------|
| grpc-request | 硬编码默认值 | 来自 `_.get(json, 'body', {...})` 的第二个参数，**不读取** `json.grpc.body` |
| ws-request | 硬编码默认值 | 来自 `_.get(json, 'body', {...})` 的第二个参数，**不读取** `json.ws.body` |
| http-request | json.http.body | 显式读取 http.body 字段设置 mode |
| graphql-request | json.http.body | 显式读取 http.body 字段设置 mode |

> **注意**：gRPC 和 WS 的 body.mode **不来自** `grpc.body` 或 `ws.body`！
> - 无 body 块时：使用硬编码默认对象，mode = 'grpc' 或 'ws'
> - 有 body:* 块时：直接使用该 body 对象，**可能缺少 mode 字段**（如 body:json 只有 json 字段）
> - `grpc.body` 和 `ws.body` 仅用于 **stringify 序列化**时写入 Bru 文件

#### 6.6.2 Body 内容分支逻辑

```javascript
if (requestType === 'grpc-request') {
  // gRPC 请求体：使用硬编码默认结构，不读取 grpc.body
  transformedJson.request.body = _.get(json, 'body', {
    mode: 'grpc',  // ← 硬编码默认值
    grpc: _.get(json, 'body.grpc', [
      { name: 'message 1', content: '{}' }
    ])
  });
} else if (requestType === 'ws-request') {
  // WebSocket 请求体：使用硬编码默认结构，不读取 ws.body
  transformedJson.request.body = _.get(json, 'body', {
    mode: 'ws',  // ← 硬编码默认值
    ws: _.get(json, 'body.ws', [
      { name: 'message 1', content: '{}' }
    ])
  });
} else {
  // HTTP / GraphQL：显式读取 http.body 设置 mode
  transformedJson.request.body = _.get(json, 'body', {});
  transformedJson.request.body.mode = _.get(json, 'http.body', 'none');
}
```

#### 6.6.3 Body 类型对照表

| Bru body 块 | 目标 JSON 字段 | 类型 |
|-------------|---------------|------|
| body:json | body.json | string |
| body:text | body.text | string |
| body:xml | body.xml | string |
| body:sparql | body.sparql | string |
| body:graphql | body.graphql.query | string |
| body:graphql:vars | body.graphql.variables | string |
| body:form-urlencoded | body.formUrlEncoded | KeyValue[] |
| body:multipart-form | body.multipartForm | MultipartForm[] |
| body:file | body.file | FileList |
| body.grpc | body.grpc | GrpcMessage[] |
| body.ws | body.ws | WebSocketMessage[] |

### 6.7 Params 字段映射（仅 HTTP/GraphQL）

| 请求类型 | 是否包含 params | 源字段 |
|----------|----------------|--------|
| http-request | 是 | json.params |
| graphql-request | 是 | json.params |
| grpc-request | 否 | - |
| ws-request | 否 | - |

```javascript
if (requestType !== 'grpc-request' && requestType !== 'ws-request') {
  (transformedJson.request as any).params = _.get(json, 'params', []);
}
```

### 6.8 gRPC 特有字段

| 字段 | 源 | 说明 |
|------|-----|------|
| methodType | json.grpc.methodType | unary / client-streaming / server-streaming / bidi-streaming |
| protoPath | json.grpc.protoPath | proto 文件路径 |

```javascript
if (requestType === 'grpc-request') {
  const selectedMethodType = _.get(json, 'grpc.methodType');
  selectedMethodType && ((transformedJson.request as any).methodType = selectedMethodType);
  const protoPath = _.get(json, 'grpc.protoPath');
  protoPath && ((transformedJson.request as any).protoPath = protoPath);
}
```

### 6.9 OAuth2 额外参数处理

当检测到 OAuth2 认证时，会收集所有额外参数并按类型分组：

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

额外参数分类：
- `authorization`：授权请求参数（仅 authorization_code 和 implicit grant type）
- `token`：令牌请求参数
- `refresh`：刷新令牌参数

每个参数包含：name, value, enabled, sendIn(headers/queryparams/body)

### 6.10 完整字段映射表

| Bru 源字段 | 目标请求字段 | 说明 |
|------------|-------------|------|
| meta.name | name | 请求名称 |
| meta.seq | seq | 排序序号，默认 1 |
| meta.tags | tags | 标签数组 |
| settings | settings | 请求设置 |
| http/grpc/ws.method | request.method | 请求方法 |
| http/grpc/ws.url | request.url | 请求 URL |
| headers/metadata | request.headers | 请求头/元数据 |
| auth.* | request.auth.* | 完整认证配置 |
| http/grpc/ws.auth | request.auth.mode | 当前选中的认证模式 |
| body.* | request.body.* | 请求体内容 |
| http.body | request.body.mode | 当前选中的 body 模式（仅 HTTP/GraphQL） |
| 硬编码默认值 | request.body.mode | gRPC/WS 的 body.mode 默认值 |
| params | request.params | 请求参数（仅 HTTP） |
| script | request.script | 脚本对象 { req, res } |
| vars | request.vars | 变量对象 { req, res } |
| assertions | request.assertions | 断言数组 |
| tests | request.tests | 测试脚本 |
| docs | request.docs | 文档描述 |
| examples | examples | 请求示例数组 |
| grpc.methodType | request.methodType | gRPC 方法类型 |
| grpc.protoPath | request.protoPath | gRPC proto 文件路径 |

### 6.11 最终输出结构对比

#### 6.11.1 http-request 结构

```typescript
interface HttpRequest {
  url: string;
  method: string;          // 大写，如 "GET", "POST"
  headers: KeyValue[];
  params: HttpRequestParam[];  // 包含 type: query | path
  auth?: Auth | null;
  body?: HttpRequestBody | null;
  script?: Script | null;
  vars?: { req: Variables; res: Variables } | null;
  assertions?: KeyValue[] | null;
  tests?: string | null;
  docs?: string | null;
}

interface HttpRequestBody {
  mode: 'none' | 'json' | 'text' | 'xml' | 'formUrlEncoded' | 'multipartForm' | 'graphql' | 'sparql' | 'file';
  json?: string | null;
  text?: string | null;
  xml?: string | null;
  sparql?: string | null;
  formUrlEncoded?: KeyValue[] | null;
  multipartForm?: MultipartForm | null;
  graphql?: GraphqlBody | null;
  file?: FileList | null;
}
```

#### 6.11.2 grpc-request 结构

```typescript
interface GrpcRequest {
  url: string;
  method?: string | null;        // gRPC 方法名，不转大写
  methodType?: GrpcMethodType | null;  // unary / streaming
  protoPath?: string | null;
  headers: KeyValue[];
  auth?: Auth | null;
  body: GrpcRequestBody;
  script?: Script | null;
  vars?: { req: Variables; res: Variables } | null;
  assertions?: KeyValue[] | null;
  tests?: string | null;
  docs?: string | null;
}

interface GrpcRequestBody {
  mode: 'grpc';
  grpc?: GrpcMessage[] | null;
}

interface GrpcMessage {
  name?: string | null;
  content?: string | null;
}
```

#### 6.11.3 ws-request 结构

```typescript
interface WebSocketRequest {
  url: string;
  headers: KeyValue[];
  auth?: Auth | null;
  body: WebSocketRequestBody;
  script?: Script | null;
  vars?: { req: Variables; res: Variables } | null;
  assertions?: KeyValue[] | null;
  tests?: string | null;
  docs?: string | null;
}

interface WebSocketRequestBody {
  mode: 'ws';
  ws?: WebSocketMessage[] | null;
}

interface WebSocketMessage {
  name?: string | null;
  type?: string | null;
  content?: string | null;
}
```

## 7. 反向转换：JSON 到 Bru（stringifyBruRequest）

### 7.1 类型反转映射

| Filestore type | Bru meta.type |
|-----------------|---------------|
| http-request | http |
| graphql-request | graphql |
| grpc-request | grpc |
| ws-request | ws |
| 其他 | http |

### 7.2 Method 字段处理

| 请求类型 | 处理逻辑 |
|----------|----------|
| grpc-request | 不转小写，保留特殊字符 |
| http/graphql | 转小写，如 "get", "post" |

### 7.3 Headers / Metadata 反转

| 请求类型 | Bru 目标块 |
|----------|-----------|
| grpc-request | metadata { ... } |
| 其他 | headers { ... } |

### 7.4 Body 反转逻辑

gRPC 和 WebSocket 请求体会被序列化为对应的 grpc/ws 块，而 HTTP 请求体会根据 mode 值序列化为对应的 body:* 块。

## 8. V1 与 V2 版本对比

### 8.1 主要差异

| 特性 | V1 | V2 |
|------|----|----|
| 语法风格 | 行内标签 + 开始/结束标签 | 块结构（类似 JSON） |
| 解析技术 | arcsecond（解析器组合子） | Ohm.js（PEG 语法） |
| 键值对格式 | 数字开头表示启用状态 | `~` 前缀表示禁用 |
| 支持的块类型 | 较少 | 丰富（支持 gRPC、WebSocket、多种认证） |
| 扩展性 | 有限 | 良好（可轻松添加新块类型） |

### 8.2 版本迁移

V2 版本完全向后兼容，推荐使用 V2 格式。

## 9. 关键技术点

### 9.1 Ohm.js 语法解析优势

1. **声明式语法**：使用 PEG 语法定义，可读性高
2. **分离的语义操作**：语法匹配与语义转换分离
3. **错误报告**：内置详细的语法错误定位
4. **可扩展性**：易于扩展新的语法规则

### 9.2 文本块处理策略

1. **缩进处理**：自动处理文本块的缩进
2. **多行文本**：支持使用 `'''` 包裹的多行文本
3. **内容类型注解**：支持 `@contentType(...)` 注解

### 9.3 键名处理

1. **特殊字符支持**：键名包含特殊字符时自动加引号
2. **转义字符**：支持 `\"` 转义引号
3. **禁用标记**：`~` 前缀标记禁用项

### 9.4 请求类型归一化设计

1. **统一输出结构**：所有请求类型共享相同的顶级结构（type, name, seq, settings, tags, request, examples）
2. **差异化处理**：通过分支逻辑处理不同请求类型的特殊字段
3. **默认值策略**：为每个请求类型设置合理的默认值，确保即使 Bru 文件不完整也能正常工作

## 10. 使用示例

### 10.1 基本用法

```javascript
const { bruToJsonV2, jsonToBruV2 } = require('@usebruno/lang');
const { parseBruRequest, stringifyBruRequest } = require('@usebruno/filestore/dist/formats/bru');

// 第一步：Bru 纯文本 转 Bru JSON
const bruContent = fs.readFileSync('request.bru', 'utf8');
const bruJson = bruToJsonV2(bruContent);

// 第二步：Bru JSON 转 可执行请求对象
const requestObj = parseBruRequest(bruJson, true);
console.log(requestObj.type);  // 'http-request', 'grpc-request', 等

// 反向转换
const bruJsonOut = stringifyBruRequest(requestObj);
const bruText = jsonToBruV2(bruJsonOut);
```

### 10.2 环境文件解析

```javascript
const { bruToEnvJsonV2 } = require('@usebruno/lang');

const envContent = fs.readFileSync('Local.bru', 'utf8');
const envJson = bruToEnvJsonV2(envContent);
```

## 11. 扩展与维护

### 11.1 添加新的块类型

1. 在 Ohm grammar 中添加语法规则
2. 在语义操作中添加转换逻辑
3. 在 filestore 的 parseBruRequest 中添加映射规则
4. 添加对应的测试用例

### 11.2 添加新的请求类型

1. 在 bruToJson.js 中添加新的块解析规则
2. 在 parseBruRequest 中添加类型分支和字段映射
3. 更新 Schema 类型定义
4. 在 stringifyBruRequest 中添加反向转换逻辑

### 11.3 常见问题排查

1. **语法错误**：检查 Ohm grammar 定义是否正确
2. **转换错误**：检查语义操作中的映射逻辑
3. **缩进问题**：确认文本块的缩进处理
4. **特殊字符**：检查键名和值中的特殊字符处理
5. **类型不匹配**：检查 filestore 层的类型映射是否正确

## 12. 总结

Bru 解析器通过 Ohm.js 提供了强大的语法解析能力，支持丰富的 HTTP 请求定义功能。从 V1 到 V2 的演进中，语法设计更加清晰、扩展性更强，为 Bruno API 客户端提供了坚实的基础。

Filestore 层的归一化处理确保了：
- 四种请求类型（http-request, graphql-request, grpc-request, ws-request）拥有统一的结构
- 各类型的特殊字段通过分支逻辑正确映射
- 合理的默认值策略保证了兼容性和健壮性
- OAuth2 额外参数的结构化分组便于后续执行层使用

主要特点：
- 基于 Ohm.js 的 PEG 语法解析
- 支持多种请求类型和认证方式
- 灵活的变量和脚本支持
- 完整的双向转换（Bru ↔ JSON）
- 良好的扩展性和可维护性
- 统一的请求对象结构便于执行层处理

---

## 附录：关键行为分析索引

### 已修正的关键事实

本文档已针对以下三处关键行为进行修正，确保所有描述与源码行为一致：

1. **HTTP/GraphQL method 默认值**：原文档表述为 HTTP 默认 GET、GraphQL 默认 POST，实际所有非 gRPC 请求默认 method 均为空字符串 `''`
2. **ws-request 的 method 字段**：原文档认为 ws-request 无 method 字段，实际所有请求类型共享统一结构，method 字段被无条件设置，最终值为空字符串
3. **gRPC/WS body.mode 来源**：原文档认为 body.mode 来自 `grpc.body`/`ws.body`，实际 mode 是硬编码在 `_.get` 的默认对象中，不读取上述字段

### 深度分析报告

完整的源码分析、执行流程验证、最小可复现示例，请参考：

**📄 `bru-parser-analysis.md`**

该报告包含：
- 每处修正的三段对照（原有说法 → 源码真实行为 → 修正后结论）
- 详细的执行流程断点分析
- 6 个最小可复现 Bru 文件示例
- 潜在问题识别与改进建议
- 源码位置索引表
