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

## 10. 三种来源字段丢失对照与处理策略

| 功能域 | Postman 导入 | OpenAPI 导入 | Insomnia 导入 | 通用处理策略 |
|-------|-------------|-------------|--------------|------------|
| **认证 (Auth)** | | | | |
| Basic | ✅ 完整支持 | ✅ 支持（变量占位符） | ✅ 完整支持 | - |
| Bearer | ✅ 完整支持 | ✅ 支持（变量占位符） | ✅ 完整支持 | - |
| Digest | ✅ 完整支持 | ✅ 支持（变量占位符） | ❌ **不支持，降级 none** | 导入后手动配置 |
| NTLM | ✅ 完整支持 | ❌ OpenAPI 无此 scheme | ❌ **不支持，降级 none** | 导入后手动配置 |
| AWS SigV4 | ✅ 完整支持 | ❌ OpenAPI 无此 scheme | ❌ **不支持，降级 none** | 导入后手动配置 |
| API Key | ✅ 支持（仅 header） | ✅ 支持（header/query） | ❌ **不支持，降级 none** | 导入后手动配置 |
| WSSE | ✅ 完整支持 | ❌ OpenAPI 无此 scheme | ❌ **不支持，降级 none** | 导入后手动配置 |
| OAuth 1.0 | ✅ 完整支持 | ❌ OpenAPI 无此 scheme | ❌ **不支持，降级 none** | 导入后手动配置 |
| OAuth 2.0 | ✅ 4 种授权流完整 | ✅ 支持（变量占位符） | ❌ **不支持，降级 none** | 导入后手动配置 |
| **脚本 (Scripts)** | | | | |
| Pre-request | ✅ 代码文本导入 | ❌ OpenAPI 无脚本概念 | ❌ **不支持** | - |
| Post-response / Tests | ✅ 代码文本导入 | ❌ OpenAPI 无脚本概念 | ❌ **不支持** | - |
| 脚本 API 语义 | ⚠️ 文本保留，不转换 `pm.*` → `bru.*` | - | - | 运行时可能出错，需手动修改 |
| **环境变量** | | | | |
| 普通变量 | ✅ 完整支持 + secret 标记 | ❌ OpenAPI 无环境概念 | ✅ 扁平化导入 | - |
| 嵌套结构变量 | ✅ 原样保留 key | ❌ - | ⚠️ **强制扁平化，点号分隔** | 嵌套结构丢失层级 |
| 初始值 | ✅ 导入（secret 标记清空） | ❌ - | ✅ 导入 | - |
| **请求体 (Body)** | | | | |
| JSON/XML/Text | ✅ 完整支持 | ⚠️ 从 Schema 生成示例值 | ✅ 完整支持 | OpenAPI 示例值可能不真实 |
| form-urlencoded | ✅ 完整支持 | ✅ 支持 | ✅ 完整支持 | - |
| multipart/form-data | ✅ 完整支持（文本） | ✅ 支持 | ⚠️ **仅文本，不支持文件** | 文件需重新选择 |
| GraphQL | ✅ 完整支持 | ✅ 支持 | ✅ 完整支持 | - |
| 二进制/文件 | ✅ 文件路径保留 | ❌ - | ❌ **不支持，丢弃** | 导入后手动添加 |
| **其他特性** | | | | |
| 示例响应 (Examples) | ✅ 完整导入 | ✅ 从 response examples 生成 | ❌ **不支持** | - |
| 文件夹排序 | ✅ seq 序号 Hydrate | ✅ 按 tags/path 分组 | ✅ 按原结构 | - |
| 变量语法 | ✅ `{{var}}` 保留 | ✅ `{{var}}` 占位符 | ⚠️ 空格/前缀清理：`{{ _.a b}}` → `{{ab}}` | 可能导致变量名变化 |
| 重复名称处理 | ✅ 自动添加 `_1, _2...` 后缀 | ✅ 自动添加后缀 | ✅ 自动添加后缀 | - |
| Cookie | ❌ **不支持** | ❌ OpenAPI cookie 参数 → header | ❌ **不支持** | Cookie 需手动管理 |
| 代理配置 | ❌ **不支持** | ❌ - | ❌ - | 独立配置，不随集合导入 |
| 证书配置 | ❌ **不支持** | ❌ - | ❌ - | 独立配置，不随集合导入 |

---

## 11. 导入导出最佳实践

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
