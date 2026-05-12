# Bruno 导入导出中间表示 (OpenCollection) 设计文档

## 1. 概述

Bruno 采用 **OpenCollection** 作为中间表示层，实现了从 Postman、OpenAPI、Insomnia 等外部格式到 Bruno 内部格式的双向转换。这一架构设计使得：

- 统一的中间格式降低了多格式支持的复杂度
- 字段映射策略保证了核心信息不丢失
- 扩展机制支持工具特有属性的往返转换

```
┌──────────────┐     ┌─────────────────┐     ┌──────────────┐
│  Postman     │────▶│                 │────▶│              │
├──────────────┤     │                 │     │              │
│  OpenAPI     │────▶│  OpenCollection │────▶│    Bruno     │
├──────────────┤     │   (中间表示)    │     │              │
│  Insomnia    │────▶│                 │────▶│              │
└──────────────┘     └─────────────────┘     └──────────────┘
```

---

## 2. 中间结构：OpenCollection 核心模型

### 2.1 顶层结构

`packages/bruno-converters/src/opencollection/types.ts`

```typescript
interface OpenCollection {
  opencollection: '1.0.0';           // 版本标识
  info: {                            // 集合元信息
    name: string;
    description?: string;
    version?: string;
  };
  items: (HttpRequest | GraphQLRequest | GrpcRequest | WebSocketRequest | Folder)[];
  config?: {                         // 配置层
    environments?: Environment[];    // 环境变量
    protobuf?: Protobuf;             // gRPC Protobuf 配置
    proxy?: Proxy;                   // 代理配置
    clientCertificates?: ClientCertificate[];
  };
  request?: RequestDefaults;         // 集合级默认请求配置
  docs?: {                           // 文档
    content: string;
    type: 'text/markdown';
  };
  bundled?: boolean;                 // 是否为捆绑格式
  extensions?: {                     // 扩展字段：存储工具特有属性
    bruno?: {
      ignore?: string[];
      presets?: {
        requestType?: string;
        requestUrl?: string;
      };
    };
  };
}
```

### 2.2 文件夹 (Folder)

`packages/bruno-converters/src/opencollection/folder.ts`

```typescript
interface Folder {
  info: {
    name: string;
    type: 'folder';
    seq?: number;                    // 排序序号
    tags?: string[];
  };
  items?: Item[];                    // 子项
  request?: RequestDefaults;         // 文件夹级默认配置
  docs?: {
    content: string;
    type: 'text/markdown';
  };
}
```

**设计要点：**
- 支持嵌套文件夹结构（递归 items）
- 继承机制：文件夹级 `request` 包含 headers、auth、scripts、variables
- 文档字段支持 Markdown 格式

### 2.3 HTTP 请求 (HttpRequest)

`packages/bruno-converters/src/opencollection/items/http.ts`

```typescript
interface HttpRequest {
  info: {
    name: string;
    type: 'http';
    seq?: number;
    tags?: string[];
  };
  http: {
    method: string;                  // GET, POST, PUT, DELETE, etc.
    url: string;
    headers?: Header[];
    params?: Param[];
    body?: HttpRequestBody;
    auth?: Auth;
  };
  runtime?: {                        // 运行时配置
    variables?: Variable[];          // 请求前变量
    scripts?: Script[];              // 脚本
    assertions?: Assertion[];        // 断言
    actions?: Action[];              // 动作（如：从响应设置变量）
  };
  settings?: {                       // 请求设置
    encodeUrl?: boolean;
    timeout?: number;
    followRedirects?: boolean;
    maxRedirects?: number;
  };
  examples?: HttpRequestExample[];   // 响应示例
  docs?: string;
}
```

### 2.4 请求体 (Body) 多态设计

`packages/bruno-converters/src/opencollection/common/body.ts`

支持多种 body 类型，通过 `type` 字段区分：

```typescript
// 原始类型
type HttpRequestBody =
  | { type: 'json'; data: string }
  | { type: 'text'; data: string }
  | { type: 'xml'; data: string }
  | { type: 'sparql'; data: string }
  | { type: 'form-urlencoded'; data: FormUrlEncodedEntry[] }
  | { type: 'multipart-form'; data: MultipartFormEntry[] }
  | { type: 'file'; data: FileBodyVariant[] };

// 表单字段结构
interface FormUrlEncodedEntry {
  name: string;
  value: string;
  description?: string;
  disabled?: boolean;
}

interface MultipartFormEntry {
  name: string;
  type: 'text' | 'file';
  value: string | string[];
  description?: string;
  disabled?: boolean;
}
```

### 2.5 认证 (Auth) 统一模型

`packages/bruno-converters/src/opencollection/common/auth.ts`

支持 9 种认证类型，通过 type 字段多态分发：

```typescript
type Auth =
  | 'inherit'                       // 继承父级
  | { type: 'basic'; username: string; password: string }
  | { type: 'bearer'; token: string }
  | { type: 'digest'; username: string; password: string }
  | { type: 'ntlm'; username: string; password: string; domain?: string }
  | { type: 'apikey'; key: string; value: string; placement: 'header' | 'query' }
  | { type: 'wsse'; username: string; password: string }
  | { type: 'awsv4'; ... }          // AWS SigV4
  | { type: 'oauth1'; ... }         // OAuth 1.0
  | { type: 'oauth2'; flow: string; ... };  // OAuth 2.0 多种授权流
```

### 2.6 环境变量 (Environment)

`packages/bruno-converters/src/opencollection/environment.ts`

```typescript
interface Environment {
  name: string;
  color?: string;
  variables: {
    name: string;
    value?: string | { data: string };  // 值或结构化值
    secret?: boolean;                    // 敏感标记
    disabled?: boolean;
  }[];
}
```

---

## 3. 字段丢失策略

### 3.1 核心原则

**策略：核心信息必保，扩展信息可选，格式差异兼容**

| 信息类别       | 丢失风险 | 处理策略                                  |
|----------------|----------|-------------------------------------------|
| URL/Method     | 低       | 强制保留，无默认值                        |
| Headers/Params | 中       | 保留所有字段，disabled 状态保持          |
| Auth 配置      | 高       | 映射到统一模型，不支持类型降级到 none    |
| 请求体         | 高       | 类型映射+内容转码，不支持类型丢弃数据    |
| 脚本           | 高       | 保留原始代码，不做语法转换                |
| 环境变量       | 中       | secret 变量导出时空值，导入时保留标记    |
| UI 顺序        | 中       | 使用 seq 字段保持排序                    |

### 3.2 认证字段映射细节

**OAuth 2.0 复杂映射：** `auth.ts:17-168`

- 授权流 (grantType)：client_credentials | password | authorization_code | implicit
- Token 放置位置：header 或 query
- Client 认证方式：body 或 basic_auth_header
- PKCE 支持：S256 方法
- 自动刷新：autoRefreshToken / autoFetchToken

**不支持的认证类型处理：**
```typescript
// 当外部格式认证类型不被 Bruno 支持时，回退到 none
default:
  return {
    mode: 'none',
    // 其他字段置 null
  };
```

### 3.3 脚本策略

`packages/bruno-converters/src/opencollection/common/scripts.ts`

```typescript
// 脚本分类存储，不做语法转换
interface Script {
  type: 'before-request' | 'after-response' | 'tests';
  code: string;
}

// Postman → Bruno 特殊处理
// 参考 postman-to-bruno.js: 只做事件类型映射，代码原样保留
// tests 合并到 after-response 中
```

**Postman 脚本转换策略（bruno-to-postman.js:216-273）：**
1. `prerequest` 事件 → Bruno `script.req`
2. `test` 事件 → Bruno `script.res` + `tests`
3. 转换失败时保留原始代码，不抛出异常

### 3.4 环境变量保密策略

`environment.ts:65-68`

```typescript
// 导出时：secret 变量值被清空
if (v.secret) {
  ocVar.secret = true;
  delete ocVar.value;  // 不导出敏感值
}

// 导入时：保留 secret 标记，值留空让用户填充
```

---

## 4. 再导出兼容策略

### 4.1 Postman 导出兼容

`packages/bruno-converters/src/postman/bruno-to-postman.js`

#### 4.1.1 URL 格式重建

```javascript
// Postman 需要结构化 URL，而非简单字符串
transformUrl(url, params) {
  // 1. 拆分 protocol, host, path
  // 2. 重建 query params 数组
  // 3. 提取 path variables（带 {var} 语法）
  return {
    raw: url,
    protocol: 'https',
    host: ['api', 'example', 'com'],
    path: ['v1', 'users'],
    query: [{ key: 'page', value: '1' }],
    variable: [{ key: 'id', value: '123' }]
  };
}
```

#### 4.1.2 集合变量推断

```javascript
// Bruno 没有显式的集合变量，导出时扫描整个集合
generateCollectionVars(collection) {
  // 1. 用正则 {{var}} 匹配所有出现的变量引用
  // 2. 提取请求/响应变量定义
  // 3. 合并去重
  // 4. 生成 Postman variable 数组（值留空）
}
```

#### 4.1.3 请求体格式映射

| Bruno Body Mode | Postman Format | 注意事项 |
|-----------------|----------------|----------|
| json/text/xml | raw + language | 设置正确的 Content-Type header |
| formUrlEncoded | urlencoded | 保留 disabled 状态 |
| multipartForm | formdata | text/file 类型区分 |
| graphql | graphql | Postman 原生支持 |
| file | file | 只保留文件路径 |

#### 4.1.4 排序策略

```javascript
// 文件夹：按名称字母序 + seq
// 请求：按 seq 序号（1, 2, 3...）
sortItemsForExport(items) {
  const folders = sortByNameThenSequence(folders);
  const requests = sortBySequence(requests);
  return [...folders, ...requests];  // 文件夹在前，请求在后
}
```

### 4.2 OpenAPI 导入策略

`packages/bruno-converters/src/openapi/openapi-to-bruno.js`

#### 4.2.1 示例值提取优先级

```javascript
// 请求体/参数示例值提取优先级（高→低）
getSchemaPropertyExampleValue(prop, propName, parentExample) {
  1. prop.example                          // 字段级示例
  2. parentExample[propName]               // 父级示例
  3. prop.default                          // 默认值
  4. prop.enum[0]                          // 枚举第一个值
  5. '' (空字符串)
}
```

#### 4.2.2 Enum 参数特殊处理

```javascript
// OpenAPI enum 参数 → Bruno 多参数条目
if (schema.enum) {
  // 为每个 enum 值创建一个参数条目
  // default 值对应的条目 enabled=true
  // 其他条目 enabled=false
}
```

#### 4.2.3 分组策略

```javascript
// 导入时可选分组方式
groupRequestsByTags()    // 按 OpenAPI tags 建文件夹
groupRequestsByPath()    // 按 URL 路径层级建文件夹
```

### 4.3 UID 处理策略

`packages/bruno-converters/src/common/index.js`

**导入时：**
- 生成新的 UUID（nanoid，不含 `-` 和 `_`）
- 每个请求、header、param、variable 都有独立 UID

**导出时：**
```javascript
deleteUidsInItems(items) {
  // 递归删除所有 UID 字段
  // 外部格式不需要 Bruno 内部 ID
}
```

---

## 5. 扩展机制与兼容性

### 5.1 OpenCollection 扩展字段

通过 `extensions` 存储 Bruno 特有属性，保证：
1. Bruno → OpenCollection → Bruno 无信息丢失
2. 其他工具导入时可以忽略这些扩展

```typescript
extensions: {
  bruno: {
    ignore: ['node_modules', '.git'],    // .gitignore 类似配置
    presets: {                           // 新建请求默认值
      requestType: 'http',
      requestUrl: 'https://api.example.com'
    }
  }
}
```

### 5.2 类型安全转换

所有转换使用 TypeScript 类型约束：
```typescript
// 入参类型检查 + 出参类型保证
fromOpenCollectionAuth(auth: Auth | undefined): BrunoAuth
toOpenCollectionAuth(auth: BrunoAuth | null | undefined): Auth | undefined
```

### 5.3 容错降级策略

**转换失败时不中断导入：**
```javascript
// 脚本转换失败回退
try {
  return translateBruToPostman(script);
} catch (err) {
  console.warn('脚本转换失败，保留原始代码', err);
  return script;  // 原样保留
}
```

---

## 6. 双向转换路径

### 6.1 Postman ↔ Bruno 完整路径

```
Postman Collection v2.1
    ↓ (postman-to-bruno.js)
Bruno Collection (JSON)
    ↓ (bruno-to-opencollection.ts)
OpenCollection (中间格式)
    ↓ (opencollection-to-bruno.ts)
Bruno Collection (验证)
    ↓ (bruno-to-postman.js)
Postman Collection v2.1 (导出)
```

### 6.2 OpenAPI → Bruno 路径

```
OpenAPI 3.0 / Swagger 2.0
    ↓ (openapi-to-bruno.js / swagger2-to-bruno.js)
    • 解析 Paths + Operations
    • 提取 Parameters (path/query/header/cookie)
    • 生成 Request Body (从 schema 推断示例)
    • 生成 Responses (示例生成)
    • 分组 (tags/path)
Bruno Collection
```

---

## 7. 已知限制与改进方向

### 7.1 信息丢失场景

| 场景 | 丢失内容 | 影响 |
|------|----------|------|
| Postman pre-request 变量引用 | 执行上下文依赖 | 脚本可能需要手动修复 |
| OAuth 2.0 令牌缓存 | access_token/expires_at | 需要重新获取令牌 |
| Postman Collection 权限配置 | 团队协作信息 | 仅导入 API 定义 |
| OpenAPI callback/webhook | 异步 API 定义 | 无法导入回调定义 |

### 7.2 潜在优化点

1. **脚本语义转换**：目前只做事件映射，不转换 API 调用（如 `pm.*` → `bru.*`）
2. **示例增强**：从 JSON Schema 生成更真实的请求/响应示例
3. **认证映射补全**：增加更多 OAuth2 自定义参数支持
4. **增量导入**：支持只导入变更部分，不覆盖已有请求

---

---

## 8. Insomnia 导入映射详解

### 8.1 Insomnia 版本支持

| 版本 | 格式标识 | 支持情况 |
|------|---------|----------|
| Insomnia v4 | `_type: 'workspace'` + resources 数组 | ✅ 完整支持 |
| Insomnia v5 | `type: 'collection.insomnia.rest/5.*'` | ✅ 完整支持 |

**双版本解析策略：**
```javascript
// packages/bruno-converters/src/insomnia/insomnia-to-bruno.js:170-176
const isInsomniaV5Export = (data) => {
  // V5 format has a type property at the root level
  if (data.type && data.type.startsWith('collection.insomnia.rest/5')) {
    return true;
  }
  return false;
};
```

### 8.2 请求核心字段映射

| Insomnia 字段 | Bruno 字段 | 处理策略 |
|--------------|-----------|----------|
| `name` | `name` | 重复名称添加后缀 `_1, _2...` |
| `url` | `request.url` | 变量语法规范化：`{{ _.var name }}` → `{{varname}}` |
| `method` | `request.method` | 原样保留 |
| `description` | `request.docs` | 描述字段 |
| `meta.id` | `uid` | 生成新 UUID |

### 8.3 Headers & Params 映射

| Insomnia 字段 | Bruno 字段 | 处理策略 |
|--------------|-----------|----------|
| `headers[].name` | `request.headers[].name` | 原样保留 |
| `headers[].value` | `request.headers[].value` | 变量语法规范化 |
| `headers[].disabled` | `request.headers[].enabled` | 取反：!disabled |
| `parameters[]` | `request.params[].type: 'query'` | Query 参数 |
| `pathParameters[]` | `request.params[].type: 'path'` | Path 参数（强制 enabled=true） |

### 8.4 认证类型映射（**仅支持 2 种**）

| Insomnia 认证类型 | Bruno 认证模式 | 说明 |
|-----------------|---------------|------|
| `basic` | `auth.mode: 'basic'` | username/password 保留，支持变量 |
| `bearer` | `auth.mode: 'bearer'` | token 保留，支持变量 |
| 其他类型 | `auth.mode: 'none'` | ⚠️ **不支持，静默降级** |

**⚠️ Insomnia 认证限制：**
- ❌ 不支持 Digest、NTLM、AWS SigV4
- ❌ 不支持 API Key（header/query 放置）
- ❌ 不支持 OAuth 1.0 / OAuth 2.0
- ❌ 不支持 WSSE
- ❌ 不支持 Hawk、Akamai、Netscape 等高级认证

### 8.5 请求体类型映射

| Insomnia MIME Type | Bruno Body Mode | 处理策略 |
|-------------------|-----------------|----------|
| `application/json` | `json` | text 内容直接赋值 |
| `application/x-www-form-urlencoded` | `formUrlEncoded` | 遍历 params 数组 |
| `multipart/form-data` | `multipartForm` | 全部设为 `type: 'text'`，⚠️ **不支持文件** |
| `text/plain` | `text` | 原样保留 |
| `text/xml` / `application/xml` | `xml` | 原样保留 |
| `application/graphql` | `graphql` | JSON 解析 query/variables |
| 其他类型 | `none` | ⚠️ **不支持，丢弃 body** |

### 8.6 环境变量特殊处理

`packages/bruno-converters/src/insomnia/env-utils.js`

**扁平化策略：**
```javascript
// Insomnia 嵌套对象结构 → Bruno 点号扁平 key
// Insomnia: { db: { host: 'localhost', port: 5432 } }
// Bruno: [ { name: 'db.host', value: 'localhost' }, { name: 'db.port', value: '5432' } ]
const flatEnvData = flattenObject(env?.data || {});
```

**子环境继承策略：**
- V4: Base env（parentId = workspaceId）+ 子 env（merge base + sub）
- V5: Base env + subEnvironments 数组（shallow merge）
- 每个子环境都生成独立的 Bruno Environment

---

## 9. 三种来源回导能力矩阵

| 来源格式 | 导入支持 | 导出支持 | 回导可行性 | 边界与原因 |
|---------|---------|---------|----------|----------|
| **Postman** | ✅ 完整 | ✅ 完整 | **双向可行** | 有 `bruno-to-postman.js`，Postman v2.1 规范完整支持 |
| **OpenAPI/Swagger** | ✅ 完整 | ❌ **不支持** | **单向导入** | 无 `bruno-to-openapi.js` 导出器；OpenAPI 是规范描述而非集合状态，导出需要 Schema 反向生成，技术复杂度高 |
| **Insomnia** | ✅ 完整 | ❌ **不支持** | **单向导入** | 无 `bruno-to-insomnia.js` 导出器；Insomnia v4/v5 格式未实现反向转换 |

### 9.1 Postman 双向转换闭环

```
Postman v2.1 Collection
    ↓ postman-to-bruno.js
Bruno Collection (JSON)
    ↓ bruno-to-postman.js
Postman v2.1 Collection (导出)
```

**导出保真度：** ~85%
- ✅ 请求元信息（URL、Method、Headers、Params）
- ✅ Body（含 form-data、x-www-form-urlencoded）
- ✅ 认证配置（Basic/Bearer/Digest/ApiKey/OAuth1/OAuth2/AWSv4）
- ✅ 文件夹结构 + 排序序号
- ✅ 示例响应（Examples）
- ⚠️ 脚本 API 调用语义不转换（仅代码文本保留）
- ⚠️ Collection Variables 需扫描推断（值可能丢失）

### 9.2 OpenAPI 单向导入边界

**为何不支持导出：**
1. **信息不对称**：OpenAPI 是 API 规范（Schema、参数定义、响应格式），Bruno 是请求集合（实际请求数据），两者语义不同
2. **无法反向生成**：从实际请求值无法反推出 JSON Schema 定义
3. **规范复杂度高**：OpenAPI 3.0 包含 Components、Security Schemes、Callbacks、Links 等高级概念，Bruno 不存储这些元数据

**导入后可导出的替代路径：**
```
OpenAPI → Bruno → [OpenCollection 中间格式] → [自定义转换] → OpenAPI
```

### 9.3 Insomnia 单向导入边界

**为何不支持导出：**
1. **实现优先级低**：社区需求较少
2. **格式差异大**：Insomnia v4 使用 resources 扁平结构 + _id 关联，v5 用嵌套 children，与 Bruno 树形结构映射成本高
3. **认证支持不对等**：Bruno 支持的认证类型远超 Insomnia 导入支持的 2 种，导出会有信息丢失

---

## 10. Postman 导入字段丢失详表

| 字段/功能 | 是否丢失 | 触发条件 | Bruno 当前处理 | 补救动作 | 最小示例 |
|----------|---------|---------|--------------|---------|---------|
| **认证类** | | | | | |
| Basic Auth | ❌ 不丢失 | - | username/password 原样导入 | - | `auth.basic.username = "admin"` → Bruno 保留 |
| Bearer Token | ❌ 不丢失 | - | token 原样导入 | - | `auth.bearer.token = "jwt_xxx"` → Bruno 保留 |
| Digest Auth | ❌ 不丢失 | - | username/password 原样导入 | - | Postman Digest → Bruno Digest 完整映射 |
| NTLM Auth | ❌ 不丢失 | - | username/password/domain 原样导入 | - | 完整支持所有字段 |
| AWS SigV4 | ❌ 不丢失 | - | accessKey/secretKey/service/region 原样导入 | - | 8 个字段完整映射 |
| API Key | ⚠️ 部分丢失 | placement = query | 仅支持 header 放置，query 丢 | 导入后改 placement | Postman `in: query` → Bruno 变 `in: header` |
| WSSE Auth | ❌ 不丢失 | - | username/password 原样导入 | - | 完整支持 |
| OAuth 1.0 | ❌ 不丢失 | - | 9 个字段完整映射 | - | consumerKey/token/signatureMethod 全保留 |
| OAuth 2.0 | ❌ 不丢失 | - | 4 种授权流完整映射 | - | client_credentials/authorization_code 等均支持 |
| **脚本类** | | | | | |
| Pre-request 脚本 | ⚠️ 文本丢失语义 | 含 `pm.*` API 调用 | 代码文本原样导入，不做 API 转换 | 手动改 `pm.*` → `bru.*` | `pm.globals.get("token")` → 导入后需改 `bru.globals.get("token")` |
| Test 脚本 | ⚠️ 文本丢失语义 | 含 `pm.test(...)` | 代码文本原样导入 | 手动改写测试逻辑 | `pm.response.to.have.status(200)` → 语法不兼容 |
| **环境变量** | | | | | |
| 普通变量 | ❌ 不丢失 | - | key/value/enabled 完整导入 | - | Postman Environment JSON → Bruno 1:1 映射 |
| Secret 变量 | ⚠️ 值丢失 | variable.type = secret | 保留 secret 标记，value 置空 | 导入后手动填值 | `{ key: "password", value: "123", type: "secret" }` → value 变空字符串 |
| Collection 变量 | ⚠️ 扫描推断 | 引用了 `{{var}}` 但未在 Environment 定义 | 正则扫描全集合提取 key，value 留空 | 导入后补充变量值 | URL 含 `{{baseUrl}}` → 推断出 collection 变量 baseUrl |
| **请求体** | | | | | |
| JSON/XML/Text | ❌ 不丢失 | - | 内容原样导入 | - | 完整保留 |
| form-urlencoded | ❌ 不丢失 | - | key/value/enabled 完整导入 | - | 所有字段保留 |
| multipart/form-data | ⚠️ 部分丢失 | param.type = file | 文件路径保留，但文件内容不导入 | 导入后重新选文件 | `{ key: "avatar", type: "file", src: "/tmp/a.jpg" }` → src 路径保留 |
| 二进制 body | ⚠️ 丢失 | body.mode = file | 文件路径保留 | 导入后重新选文件 | 路径保留，需重选 |
| **其他特性** | | | | | |
| 示例响应 Examples | ❌ 不丢失 | - | name/status/headers/body 完整导入 | - | Postman response examples → Bruno examples 完整映射 |
| Cookie | ❌ 完全丢失 | 任何 Cookie 配置 | 不解析 Cookie 字段 | 手动复制 Cookie 值到 Header | Postman cookie jar → 不导入 |
| 代理配置 | ❌ 完全丢失 | 任何代理设置 | 不导入代理配置 | Bruno 设置里手动配 | Postman proxy → 丢弃 |
| 客户端证书 | ❌ 完全丢失 | 任何证书配置 | 不导入证书 | Bruno 设置里手动配 | Postman client cert → 丢弃 |

---

## 11. OpenAPI 导入字段丢失详表

| 字段/功能 | 是否丢失 | 触发条件 | Bruno 当前处理 | 补救动作 | 最小示例 |
|----------|---------|---------|--------------|---------|---------|
| **认证类** | | | | | |
| Basic Auth | ⚠️ 丢失值 | securitySchemes.type = http, scheme = basic | 生成 `{{username}}` / `{{password}}` 占位符 | 导入后在环境变量填真实值 | OpenAPI securitySchemes → Bruno auth.basic 仅占位，无真实值 |
| Bearer Token | ⚠️ 丢失值 | securitySchemes.type = http, scheme = bearer | 生成 `{{token}}` 占位符 | 导入后填真实 token | `token: "{{token}}"` → 无真实值 |
| API Key | ⚠️ 丢失值 | securitySchemes.type = apiKey | 在 Header/Query 添加占位符 `{{apiKey}}` | 导入后填真实值 | `X-API-Key: "{{apiKey}}"` → 无真实值 |
| OAuth 2.0 | ⚠️ 部分丢失 | securitySchemes.type = oauth2 | 仅识别 grantType，其他 OAuth 配置丢 | 手动补全 OAuth 配置 | OpenAPI flows.authorizationCode → Bruno 仅 grantType，其他字段空 |
| Digest/NTLM/WSSE/OAuth1 | ❌ 完全丢失 | 任何非标准 http 认证 | 无对应 OpenAPI scheme，直接丢弃 | 导入后手动配 | OpenAPI 无 Digest scheme → Bruno auth.mode = inherit |
| **脚本类** | | | | | |
| Pre-request 脚本 | ❌ 完全丢失 | - | OpenAPI 无脚本概念 | 手动写脚本 | - |
| Test 脚本 | ❌ 完全丢失 | - | OpenAPI 无测试概念 | 手动写测试 | - |
| **环境变量** | | | | | |
| 服务器变量 | ⚠️ 丢失值 | servers[].variables | 变量名识别，值留空 | 导入后填 server 变量值 | `servers: [{ url: "{env}.api.com", variables: { env: { default: "prod" } } }]` → 只识别变量名 |
| 普通环境变量 | ❌ 完全丢失 | - | OpenAPI 无环境概念 | 手动创建环境 | - |
| **请求体** | | | | | |
| JSON Body | ⚠️ 丢失真实值 | requestBody.content["application/json"] | 从 schema.example 或 properties 生成示例值 | 替换为真实请求数据 | Schema `type: object, properties: { id: { type: number } }` → 生成 `{ "id": 0 }` 示例 |
| form-urlencoded | ⚠️ 丢失真实值 | requestBody.content["application/x-www-form-urlencoded"] | 从 schema 生成示例值 | 替换为真实数据 | 每个字段用 type 默认值（string→"", number→0） |
| multipart/form-data | ⚠️ 丢失真实值 | requestBody.content["multipart/form-data"] | 从 schema 生成示例值，不生成文件 | 手动添加文件字段 | 文件字段只生成 key，无文件路径 |
| 二进制 body | ❌ 完全丢失 | requestBody.content["application/octet-stream"] | 不处理二进制 | 手动改 body mode = file | - |
| **其他特性** | | | | | |
| 示例响应 Examples | ⚠️ 部分丢失 | responses[].content[].examples | 导入 name + body，headers/status 可能丢失 | 手动补全响应状态码和头 | OpenAPI response examples → Bruno examples 仅保留 body |
| Path 参数 | ⚠️ 丢失值 | parameters[].in = path | 参数名保留，值留空 | 导入后填参数值 | `/users/{id}` → params.name = "id", value = "" |
| Query 参数 | ⚠️ 丢失值 | parameters[].in = query | 参数名保留，从 schema 默认值/枚举生成示例 | 替换为真实值 | schema.default = 1 → value = "1" |
| Header 参数 | ⚠️ 丢失值 | parameters[].in = header | 参数名保留，从 schema 生成示例 | 替换为真实值 | - |
| Cookie 参数 | ⚠️ 转 Header 丢语义 | parameters[].in = cookie | 转成 Header 字段，Cookie 语义丢失 | 理解为普通 Header | Cookie: session=xxx → 变成 Header 名为 Cookie |
| 枚举参数 | ⚠️ 结构丢失 | schema.enum 存在 | 仅导入第一个 enum 值为默认，其他 enum 丢 | Bruno 不支持参数多选 | `enum: ["a", "b", "c"]` → 仅 "a" 为值，b/c 丢失 |
| 服务器分组 | ❌ 完全丢失 | servers 数组有多个 | 仅用第一个 server URL，其他丢 | 手动创建环境区分多服务器 | `servers: [{url: "prod.com"}, {url: "dev.com"}]` → 只用 prod.com |

---

## 12. Insomnia 导入字段丢失详表

| 字段/功能 | 是否丢失 | 触发条件 | Bruno 当前处理 | 补救动作 | 最小示例 |
|----------|---------|---------|--------------|---------|---------|
| **认证类** | | | | | |
| Basic Auth | ❌ 不丢失 | authentication.type = basic | username/password 原样导入 | - | 完整支持 |
| Bearer Token | ❌ 不丢失 | authentication.type = bearer | token 原样导入 | - | 完整支持 |
| Digest/NTLM/AWS/ApiKey/WSSE/OAuth1/OAuth2 | ❌ 完全丢失 | 任何非 basic/bearer 认证 | 静默降级为 auth.mode = none | 导入后手动重新配置认证 | authentication.type = oauth2 → Bruno auth.mode = none，所有 OAuth 字段全丢 |
| **脚本类** | | | | | |
| Pre-request 脚本 | ❌ 完全丢失 | - | Insomnia 插件/钩子不导入 | 手动写脚本 | Insomnia Request Hooks → 不导入 |
| Test 脚本 | ❌ 完全丢失 | - | Insomnia 测试不导入 | 手动写测试 | - |
| **环境变量** | | | | | |
| 普通变量 | ⚠️ 结构丢失 | 环境变量值是嵌套对象 | 强制扁平化，点号分隔 key | 检查变量引用是否需改名称 | Insomnia: `{ db: { host: "localhost" } }` → Bruno: `name = "db.host", value = "localhost"` |
| 子环境继承 | ⚠️ 结构丢失 | baseEnv + subEnvironments | Shallow merge base + sub，生成独立 Environment | 理解继承关系已固化为独立环境 | Base: { a: 1 }, Sub: { b: 2 } → Bruno Sub env: { a: 1, b: 2 } |
| Secret 标记 | ❌ 完全丢失 | - | Insomnia Secret 不识别，全部当普通变量 | 手动标记 secret 变量 | - |
| **请求体** | | | | | |
| JSON/XML/Text | ❌ 不丢失 | - | 内容原样导入 | - | 完整保留 |
| form-urlencoded | ❌ 不丢失 | - | key/value/enabled 完整导入 | - | 所有字段保留 |
| multipart/form-data | ⚠️ 严重丢失 | params[].type = file | 强制转 `type: 'text'`，文件名/路径全丢 | 导入后重新选文件 | Insomnia file param → Bruno type 变 text，value 为空字符串 |
| GraphQL | ❌ 不丢失 | body.mimeType = application/graphql | JSON 解析 query/variables | - | 完整支持 |
| 二进制 body | ❌ 完全丢失 | 其他 MIME type | body.mode 设为 none，内容全丢 | 导入后手动选文件 | 二进制/图片/压缩包 → 全部丢弃 |
| **其他特性** | | | | | |
| 示例响应 Examples | ❌ 完全丢失 | - | Insomnia Response 不导入 | 手动添加示例 | Insomnia 保存的响应 → 不导入 |
| 变量名规范化 | ⚠️ 名称变化 | 变量名含空格或 `_.` 前缀 | 去空格 + 去 `_.` 前缀 | 检查所有 `{{var}}` 引用一致性 | `{{ _.api token}}` → `{{apitoken}}` |
| 请求/文件夹名称 | ⚠️ 名称变化 | 同级别有重名 | 自动加 `_1, _2...` 后缀 | 检查文件夹/请求名称变化 | 两个 "Get User" → 变 "Get User" + "Get User_1" |
| Cookie | ❌ 完全丢失 | - | Insomnia Cookie Jar 不导入 | 手动复制 Cookie 值 | - |
| 代理配置 | ❌ 完全丢失 | - | 不导入 | 手动配置 | - |
| 客户端证书 | ❌ 完全丢失 | - | 不导入 | 手动配置 | - |
| 环境颜色 | ❌ 完全丢失 | - | Insomnia Environment color 字段不导入 | 手动改环境颜色 | - |

---

## 13. 导入导出最佳实践

### 11.1 Postman 往返导入
1. **导出前备份**：复杂集合建议先备份原始 Postman JSON
2. **脚本检查**：含 `pm.*` API 调用的脚本导入后需测试运行
3. **认证重配**：OAuth 2.0 token 等敏感信息需重新获取
4. **变量核对**：集合级变量导出时是扫描推断，可能有遗漏

### 11.2 OpenAPI 导入注意
1. **示例优先**：优先导入带完整 examples 的 OpenAPI 文档
2. **分组策略选择**：tags 分组适合业务，path 分组适合 RESTful API
3. **认证占位符**：导入后所有认证字段是 `{{变量}}`，需配置环境变量
4. **Body 真实性**：从 Schema 生成的请求体是示例值，需替换为真实数据

### 11.3 Insomnia 导入注意
1. **认证重配**：除 Basic/Bearer 外，其他认证全部丢失，需手动设置
2. **文件重选**：multipart/form-data 中的文件附件需重新选择
3. **环境变量检查**：嵌套结构被扁平化后，检查 key 名称是否正确
4. **变量名变化**：Insomnia 的 `{{ _.my var }}` 会变成 `{{myvar}}`，检查引用一致性

---

## 附：核心文件清单

| 文件路径 | 职责 |
|---------|------|
| `packages/bruno-converters/src/opencollection/types.ts` | 类型定义导出入口 |
| `packages/bruno-converters/src/opencollection/opencollection-to-bruno.ts` | OC → Bruno 主转换 |
| `packages/bruno-converters/src/opencollection/bruno-to-opencollection.ts` | Bruno → OC 主转换 |
| `packages/bruno-converters/src/opencollection/items/http.ts` | HTTP 项转换 |
| `packages/bruno-converters/src/opencollection/common/auth.ts` | 认证转换 |
| `packages/bruno-converters/src/opencollection/common/body.ts` | 请求体转换 |
| `packages/bruno-converters/src/opencollection/folder.ts` | 文件夹转换 |
| `packages/bruno-converters/src/opencollection/environment.ts` | 环境转换 |
| `packages/bruno-converters/src/postman/postman-to-bruno.js` | Postman 导入 |
| `packages/bruno-converters/src/postman/bruno-to-postman.js` | Postman 导出 |
| `packages/bruno-converters/src/postman/postman-env-to-bruno-env.js` | Postman 环境导入 |
| `packages/bruno-converters/src/openapi/openapi-to-bruno.js` | OpenAPI 3.0 导入 |
| `packages/bruno-converters/src/openapi/swagger2-to-bruno.js` | Swagger 2.0 导入 |
| `packages/bruno-converters/src/insomnia/insomnia-to-bruno.js` | Insomnia 导入（v4/v5） |
| `packages/bruno-converters/src/insomnia/env-utils.js` | Insomnia 环境转换 |
| `packages/bruno-converters/src/common/index.js` | 通用工具（UUID、schema 验证等） |
