# Bruno 导入映射规则分析

本文档分析了 Bruno 将 OpenAPI/Swagger 和 Postman 集合导入为内部结构时的映射规则，重点关注集合、请求、鉴权信息和变量的转换方式。

---

## 一、集合 (Collection) 转换

### 1.1 Postman Collection 转换

**文件位置**: `packages/bruno-converters/src/postman/postman-to-bruno.js`

#### 基本结构映射

| Postman 字段 | Bruno 字段 | 说明 |
|------------|-----------|------|
| `info.name` | `name` | 集合名称，默认 "Untitled Collection" |
| - | `uid` | 自动生成 UUID |
| - | `version` | 固定为 "1" |
| `item` | `items` | 递归转换为文件夹或请求项 |
| `variable` | `root.request.vars.req` | 集合级变量 |
| `event` | `root.request.script` | 集合级脚本 |
| `auth` | `root.request.auth` | 集合级鉴权配置 |

#### 集合级鉴权处理

```javascript
// 代码位置: processAuth 函数 (line 219)
- 当 auth 为 null 时，集合保持默认 mode = 'none'
- 支持的鉴权类型: basic, bearer, awsv4, apikey, digest, oauth1, oauth2
- 不支持的类型默认设置为 mode = 'none'
```

#### 集合级变量处理

```javascript
// 代码位置: importCollectionLevelVariables 函数 (line 208)
- 过滤掉 key 和 value 均为 null 的变量
- 变量名替换非法字符: invalidVariableCharacterRegex
- 变量值如果是对象会被 JSON.stringify
- 所有变量默认 enabled = true
```

### 1.2 OpenAPI/Swagger 转换

**文件位置**:
- `packages/bruno-converters/src/openapi/openapi-to-bruno.js`
- `packages/bruno-converters/src/openapi/swagger2-to-bruno.js`

#### 基本结构映射

| OpenAPI 字段 | Bruno 字段 | 说明 |
|-------------|-----------|------|
| `info.title` | `name` | 集合名称，默认 "Untitled Collection" |
| - | `uid` | 自动生成 UUID |
| - | `version` | 固定为 "1" |
| `paths` | `items` | 按标签或路径分组为文件夹和请求 |
| `servers` | `environments` | 每个 server 创建一个环境 |
| `security` / `securitySchemes` | `root.request.auth` | 集合级鉴权配置 |

#### 环境 (Environments) 生成

```javascript
// OpenAPI 3.x (line 818)
- 每个 servers 条目创建一个 Environment
- 环境变量包含 baseUrl 和服务器变量（带默认值）
- 服务器变量会被替换为模板变量 {{variableName}}

// Swagger 2.0 (line 559)
- 从 host, basePath, schemes 构建 server URL
- 每个 scheme 创建一个 Environment（如 http, https）
- 仅包含 baseUrl 变量
```

---

## 二、文件夹 (Folder) 转换

### 2.1 Postman Folder 转换

#### 判定规则

```javascript
// 代码位置: isItemAFolder 函数 (line 73)
- Postman item 没有 request 字段即为文件夹
```

#### 结构映射

| Postman 字段 | Bruno 字段 | 说明 |
|------------|-----------|------|
| `name` | `name` | 文件夹名称，重名会自动加序号 |
| - | `uid` | 自动生成 UUID |
| - | `type` | 固定为 "folder" |
| - | `seq` | 序列顺序号 |
| `description` | `root.docs` | 文件夹描述 |
| `auth` | `root.request.auth` | 文件夹级鉴权，默认 mode = 'inherit' |
| `event` | `root.request.script` | 文件夹级脚本 |
| 嵌套 `item` | `items` | 递归处理子文件夹和请求 |

### 2.2 OpenAPI/Swagger 分组

支持两种分组方式（通过 `options.groupBy` 配置）:

#### 方式一: 按标签分组 (tags) - 默认

```javascript
// 代码位置: groupRequestsByTags 函数 (openapi-common.js line 504)
- 取第一个 tag 作为文件夹名称
- 没有标签的请求放入根级别（ungrouped）
- 标签会经过 sanitizeTag 清洗
```

#### 方式二: 按路径分组 (path)

```javascript
// 代码位置: groupRequestsByPath 函数 (openapi-common.js line 542)
- 按 URL 路径段创建嵌套文件夹结构
- 例如 /api/v1/users -> api > v1 > users
- 参数路径段（如 {id}）也作为文件夹名称
```

---

## 三、请求 (Request) 转换

### 3.1 Postman Request 转换

#### 基本信息映射

| Postman 字段 | Bruno 字段 | 说明 |
|------------|-----------|------|
| `name` | `name` | 请求名称，重名自动加序号 |
| - | `uid` | 自动生成 UUID |
| - | `type` | "http-request" 或 "graphql-request" |
| `request.method` | `request.method` | HTTP 方法，转大写 |
| `request.url` | `request.url` | 完整 URL，移除 #fragment |
| `request.description` | `request.docs` | 请求描述 |

#### URL 构建

```javascript
// 代码位置: constructUrl / constructUrlFromParts (line 151)
- 优先使用 url.raw（移除 #fragment）
- raw 不存在时，从 protocol + host + path + port + query 构建
- host 数组用 . 连接，path 数组用 / 连接
```

#### Header 转换

```javascript
// 代码位置: normalizeHeaders (line 111)
- 支持多种 header 格式: 对象数组、字符串数组、单个换行分隔字符串
- 字符串格式: "Key: Value" 解析为键值对
- 值非字符串会被转换: 数字直接转字符串，对象 JSON.stringify
```

#### Query / Path Params 转换

```javascript
// Query 参数 (line 611)
- 遍历 request.url.query 数组
- 每个参数创建: uid, name, value, description, type='query', enabled=true
- key/value 均为 null 时跳过

// Path 参数 (line 625)
- 遍历 request.url.variable 数组
- 每个参数创建: uid, name, value, description, type='path', enabled=true
- 无 key 时跳过该参数
```

#### Body 转换

| Postman body.mode | Bruno body.mode | 处理逻辑 |
|------------------|-----------------|---------|
| `formdata` | `multipartForm` | 文件类型（type=file 或 type=default+src）设为 type='file'，否则 'text' |
| `urlencoded` | `formUrlEncoded` | 键值对形式存储 |
| `raw` + language=json | `json` | 存储原始字符串 |
| `raw` + language=xml | `xml` | 存储原始字符串 |
| `raw` + 其他 | `text` | 存储原始字符串 |
| `graphql` | `graphql` | 提取 query 和 variables |

#### Script 转换

```javascript
// 代码位置: importScriptsFromEvents (line 176)
- prerequest 事件 -> script.req
- test 事件 -> script.res
- 通过 postmanTranslation 函数转换 Postman 脚本语法
- 支持 Worker 异步处理以避免阻塞
```

#### Response Examples 转换

```javascript
// 代码位置: line 642-802
- 遍历 item.response 数组
- 每个响应创建一个 Example 对象
- 包含: 状态码、状态文本、响应头、响应体（按 Content-Type 解析）
- 同时保留原始请求信息（URL、方法、头、参数、Body）
```

### 3.2 OpenAPI/Swagger Request 转换

#### 基本信息映射

| OpenAPI 字段 | Bruno 字段 | 说明 |
|-------------|-----------|------|
| `summary` / `operationId` / `description` | `name` | 优先级: summary > operationId > description > "METHOD path" |
| - | `uid` | 自动生成 UUID |
| - | `type` | "http-request" |
| `tags` | `tags` | 请求标签 |
| HTTP Method | `request.method` | 转大写 |
| Path + Server | `request.url` | {{baseUrl}} + path |
| `description` | `request.docs` | 请求描述 |

#### 路径参数处理

```javascript
// OpenAPI 3.x (line 191)
- {param} 替换为 {{operationId_param}} 格式
- 例如 /users/{id} -> /users/{{getUserById_id}}

// Swagger 2.0 (line 151)
- {param} 替换为 :param 格式
- 例如 /users/{id} -> /users/:id
```

#### Parameters 转换

```javascript
// 参数类型: query, path, header
// 代码位置: transformOpenapiRequestItem (line 242-327)

- 支持枚举(enum)参数: 每个枚举值创建一个参数条目
- 支持对象类型参数: 展开 properties 为独立参数
- 值优先级: example > default > enum[0] > ''
- enabled: 必填参数或有值时为 true
```

#### Request Body 转换

通过 `BODY_TYPE_HANDLERS` 处理不同 Content-Type:

| Content-Type Pattern | Bruno body.mode | 处理逻辑 |
|---------------------|-----------------|---------|
| `*/json`, `*/*+json` | `json` | 从 schema 生成示例 JSON |
| `application/x-www-form-urlencoded` | `formUrlEncoded` | 从 schema properties 生成 |
| `multipart/form-data` | `multipartForm` | format=binary 的字段设为 type='file' |
| `*/xml`, `*/*+xml` | `xml` | 从 schema 生成示例 XML |
| `application/sparql-query` | `sparql` | 直接使用 example 值 |
| `text/*`, `application/octet-stream`, `*/*` | `text` | 直接使用 example 值 |

### 3.3 Request Examples 生成

```javascript
// 代码位置: createBrunoExample (openapi-common.js line 433)
- 从 responses 提取示例
- 支持 response.examples 和 response.example
- 无示例时从 schema 自动生成
- 每个示例包含完整的请求和响应快照
- 支持匹配 request body 和 response example 的 key
```

---

## 四、鉴权 (Auth) 转换

### 4.1 Postman Auth 转换

**代码位置**: `processAuth` 函数 (postman-to-bruno.js line 219)

#### 鉴权类型映射

| Postman Auth Type | Bruno Auth Mode |
|------------------|-----------------|
| `basic` | `basic` |
| `bearer` | `bearer` |
| `awsv4` | `awsv4` |
| `apikey` | `apikey` |
| `digest` | `digest` |
| `oauth1` | `oauth1` |
| `oauth2` | `oauth2` |
| `noauth` | `none` |
| `null` (集合级) | 保持默认 'none' |
| `null` (文件夹/请求级) | 保持默认 'inherit' |

#### Basic Auth 映射

| Postman 字段 | Bruno 字段 |
|------------|-----------|
| `username` | `basic.username` |
| `password` | `basic.password` |

#### Bearer Auth 映射

| Postman 字段 | Bruno 字段 |
|------------|-----------|
| `token` | `bearer.token` |

#### API Key Auth 映射

| Postman 字段 | Bruno 字段 | 说明 |
|------------|-----------|------|
| `key` | `apikey.key` | - |
| `value` | `apikey.value` | - |
| `in` | `apikey.placement` | 'query' -> 'queryparams', 其他 -> 'header' |

#### AWSv4 Auth 映射

| Postman 字段 | Bruno 字段 |
|------------|-----------|
| `accessKey` | `awsv4.accessKeyId` |
| `secretKey` | `awsv4.secretAccessKey` |
| `sessionToken` | `awsv4.sessionToken` |
| `service` | `awsv4.service` |
| `region` | `awsv4.region` |

#### OAuth1 Auth 映射

| Postman 字段 | Bruno 字段 |
|------------|-----------|
| `consumerKey` | `oauth1.consumerKey` |
| `consumerSecret` | `oauth1.consumerSecret` |
| `token` | `oauth1.accessToken` |
| `tokenSecret` | `oauth1.accessTokenSecret` |
| `callback` | `oauth1.callbackUrl` |
| `verifier` | `oauth1.verifier` |
| `signatureMethod` | `oauth1.signatureMethod` |
| `privateKey` | `oauth1.privateKey` |
| `timestamp` | `oauth1.timestamp` |
| `nonce` | `oauth1.nonce` |
| `version` | `oauth1.version` |
| `realm` | `oauth1.realm` |
| `addParamsToHeader` | `oauth1.placement` |

#### OAuth2 Auth 映射

| Postman Grant Type | Bruno Grant Type |
|------------------|-----------------|
| `authorization_code` | `authorization_code` |
| `authorization_code_with_pkce` | `authorization_code` |
| `password_credentials` | `password` |
| `client_credentials` | `client_credentials` |

| Postman 字段 | Bruno 字段 |
|------------|-----------|
| `accessTokenUrl` | `oauth2.accessTokenUrl` |
| `refreshTokenUrl` | `oauth2.refreshTokenUrl` |
| `clientId` | `oauth2.clientId` |
| `clientSecret` | `oauth2.clientSecret` |
| `scope` | `oauth2.scope` |
| `state` | `oauth2.state` |
| `addTokenTo` | `oauth2.tokenPlacement` |
| `client_authentication` | `oauth2.credentialsPlacement` |

### 4.2 OpenAPI/Swagger Auth 转换

#### Security Scheme 类型映射

| OpenAPI Type | Scheme | Bruno Auth Mode |
|-------------|--------|-----------------|
| `http` | `basic` | `basic` |
| `http` | `bearer` | `bearer` |
| `http` | `digest` | `digest` |
| `apiKey` | - | `apikey` |
| `oauth2` | - | `oauth2` |

#### Basic/Bearer/Digest Auth

```javascript
// 所有凭证值均设为模板变量
- username: {{username}}
- password: {{password}}
- token: {{token}}
```

#### API Key Auth

| OpenAPI 字段 | Bruno 字段 |
|-------------|-----------|
| `name` | `apikey.key` |
| - | `apikey.value` | {{apiKey}} |
| `in` | `apikey.placement` | 'query' -> 'queryparams', 其他 -> 'header' |

**注意**: API Key 同时会被添加到 headers 或 params 中。

#### OAuth2 Auth 映射

| OpenAPI Flow | Bruno Grant Type |
|-------------|-----------------|
| `authorizationCode` | `authorization_code` |
| `implicit` | `implicit` |
| `password` | `password` |
| `clientCredentials` | `client_credentials` |

| Swagger 2.0 Flow | Bruno Grant Type |
|-----------------|-----------------|
| `accessCode` | `authorization_code` |
| `implicit` | `implicit` |
| `password` | `password` |
| `application` | `client_credentials` |

**OAuth2 字段映射**:

| OpenAPI/Swagger 字段 | Bruno 字段 |
|---------------------|-----------|
| `authorizationUrl` | `oauth2.authorizationUrl` |
| `tokenUrl` | `oauth2.accessTokenUrl` |
| `refreshUrl` | `oauth2.refreshTokenUrl` |
| `scopes` | `oauth2.scope` (空格分隔) |
| - | `oauth2.callbackUrl` | {{oauth_callback_url}} |
| - | `oauth2.clientId` | {{oauth_client_id}} |
| - | `oauth2.clientSecret` | {{oauth_client_secret}} |
| - | `oauth2.state` | {{oauth_state}} |
| - | `oauth2.credentialsPlacement` | 'header' |
| - | `oauth2.tokenPlacement` | 'header' |
| - | `oauth2.tokenHeaderPrefix` | 'Bearer' |
| - | `oauth2.autoFetchToken` | false |
| - | `oauth2.autoRefreshToken` | true |

---

## 五、变量 (Variables) 转换

### 5.1 Postman Variables

**代码位置**: `importCollectionLevelVariables` (postman-to-bruno.js line 208)

#### 集合级变量

| Postman Variable 字段 | Bruno Variable 字段 |
|---------------------|-------------------|
| `key` | `name` | 经过非法字符替换 |
| `value` | `value` | 对象会被 JSON.stringify |
| - | `uid` | 自动生成 |
| - | `enabled` | true |

### 5.2 OpenAPI Server Variables

**代码位置**: `extractServerVars` (openapi-to-bruno.js line 746)

```javascript
// Server URL 模板变量替换
- 例如: https://{tenant}.api.example.com/{version}
- 转换为: https://{{tenant}}.api.example.com/{{version}}

// 变量创建
- baseUrl: 模板化后的完整 URL
- 各 server 变量: 使用 default 值或 enum[0]
- 所有变量 type='text', enabled=true, secret=false
```

### 5.3 Request-level Server Variables

```javascript
// OpenAPI 3.x (line 228-240)
- 当 operation 或 pathItem 有独立 servers 配置时
- 服务器变量作为 request.vars.req 创建
- 变量 local=false（非本地变量）
```

### 5.4 OpenAPI Links → 脚本变量

```javascript
// 代码位置: line 463-479
- 解析 response.links 配置
- 为每个 link 生成响应脚本
- 使用 bru.setVar() 设置变量: {{operationId_parameter}}
- 变量值从响应体中提取: res.body.xxx
```

---

## 六、通用处理流程

### 6.1 导入流水线

```
原始数据
    ↓
[JSON/YAML 解析]
    ↓
[Schema 版本检测]
    ↓
[$ref 引用解析]
    ↓
[集合/文件夹/请求转换]
    ↓
[transformItemsInCollection] → 项目转换
    ↓
[hydrateSeqInCollection] → 填充序列号
    ↓
[validateSchema] → 模式校验
    ↓
Bruno 集合
```

### 6.2 命名冲突处理

```javascript
// Postman (line 377-380, 444-447)
- 同名文件夹: FolderName_1, FolderName_2...
- 同名请求: RequestName_1, RequestName_2...

// OpenAPI (line 174-187)
- 先尝试追加 "(METHOD)" 区分
- 仍冲突则追加序号: Name (1), Name (2)...
```

### 6.3 描述字段处理

```javascript
// 代码位置: transformDescription (line 57)
- 支持字符串和对象两种格式
- 对象格式取 content 字段值
- null/undefined 转换为空字符串
```

---

## 七、关键差异总结

### 7.1 Postman vs OpenAPI 导入差异

| 特性 | Postman | OpenAPI/Swagger |
|----|---------|----------------|
| 环境变量 | 需单独导入 Environment 文件 | 直接从 servers 生成 Environments |
| 鉴权继承 | 支持 inherit 模式 | 集合级设置，请求级覆盖 |
| Body 示例 | 直接使用 raw 内容 | 从 schema/example 自动生成 |
| 脚本 | 完整的 test/prerequest 脚本 | 仅 Links 生成变量提取脚本 |
| 分组 | 按文件夹层级 | 按 tags 或 path 分组 |

### 7.2 OpenAPI 3.x vs Swagger 2.0 差异

| 特性 | OpenAPI 3.x | Swagger 2.0 |
|----|-------------|-------------|
| Server 配置 | `servers` 数组 | `host` + `basePath` + `schemes` |
| 安全配置 | `components.securitySchemes` | `securityDefinitions` |
| 参数位置 | 支持 cookie | 不支持 cookie |
| Body 定义 | `requestBody.content` | `in: body` 参数 |
| 媒体类型 | 完整 MIME 类型匹配 | `consumes` / `produces` |
| OAuth2 配置 | 多 flow 支持 | 单 flow 定义 |

---

## 八、代码文件索引

| 功能 | 文件路径 | 核心函数 |
|-----|---------|---------|
| Postman 转换 | `packages/bruno-converters/src/postman/postman-to-bruno.js` | `postmanToBruno`, `processAuth`, `importScriptsFromEvents` |
| OpenAPI 3.x 转换 | `packages/bruno-converters/src/openapi/openapi-to-bruno.js` | `openApiToBruno`, `parseOpenApiCollection`, `transformOpenapiRequestItem` |
| Swagger 2.0 转换 | `packages/bruno-converters/src/openapi/swagger2-to-bruno.js` | `swagger2ToBruno`, `parseSwagger2Collection`, `transformSwaggerRequestItem` |
| 通用工具 | `packages/bruno-converters/src/openapi/openapi-common.js` | `BODY_TYPE_HANDLERS`, `createBrunoExample`, `groupRequestsByTags`, `groupRequestsByPath` |
| 环境转换 | `packages/bruno-converters/src/postman/postman-env-to-bruno-env.js` | - |
| 脚本翻译 | `packages/bruno-converters/src/postman/postman-translations.js` | - |
