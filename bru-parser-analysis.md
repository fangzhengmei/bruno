# Bru 解析器关键行为分析报告

## 概述

本报告针对 Bruno API 客户端中 Bru 格式解析器的三处关键行为进行深度源码分析与验证，纠正原有文档中的表述偏差。

**分析对象**：`packages/bruno-filestore/src/formats/bru/index.ts` 第 12-116 行 `parseBruRequest` 函数

---

## 分析一：HTTP/GraphQL method 默认值

### 1.1 文档原有说法

| 请求类型 | 源字段 | 处理逻辑 | 默认值 |
|----------|--------|----------|--------|
| http-request | json.http.method | 转大写 | GET |
| graphql-request | json.http.method | 转大写 | POST |

### 1.2 源码真实行为

**关键代码**（第 49-52 行）：
```javascript
method:
  requestType === 'grpc-request'
    ? _.get(json, 'grpc.method', '')
    : String(_.get(json, 'http.method') ?? '').toUpperCase()
```

**执行流程分析**：
1. 非 gRPC 请求（包括 HTTP、GraphQL、WS）都走 else 分支
2. `_.get(json, 'http.method')` → 当 Bru 文件无 `http { method: ... }` 定义时返回 `undefined`
3. `undefined ?? ''` → nullish coalescing 返回空字符串 `''`
4. `String('').toUpperCase()` → 结果仍为 `''`

**验证断点**：
- 输入：`http { url: "..." }`（无 method 定义）
- 中间值：`_.get(json, 'http.method')` = `undefined`
- 最终值：`String(undefined ?? '').toUpperCase()` = `''`

### 1.3 修正后结论

| 请求类型 | 默认值 | 说明 |
|----------|--------|------|
| http-request | 空字符串 `''` | 无 GET 预设 |
| graphql-request | 空字符串 `''` | 无 POST 预设 |
| grpc-request | 空字符串 `''` | 从 `grpc.method` 读取，默认 `''` |
| ws-request | 空字符串 `''` | 走非 gRPC 分支 |

### 1.4 最小可复现示例

**输入 Bru 文件**：
```
meta {
  name: Method Default Test
  type: http
}
http {
  url: https://api.example.com/test
}
```

**解析命令**：
```javascript
const result = parseBruRequest(bruContent);
console.log('method:', JSON.stringify(result.request.method));
```

**预期输出**：
```
method: ""
```

---

## 分析二：ws-request 是否包含 method 字段

### 2.1 文档原有说法

| 请求类型 | 源字段 | 处理逻辑 | 默认值 |
|----------|--------|----------|--------|
| ws-request | 无 | 无 method 字段 | - |

### 2.2 源码真实行为

**关键代码**（第 41-62 行）：
```javascript
const transformedJson = {
  type: requestType,
  name: _.get(json, 'meta.name'),
  // ...
  request: {
    // 第 47-52 行：所有请求类型都无条件执行这段代码
    method:
      requestType === 'grpc-request'
        ? _.get(json, 'grpc.method', '')
        : String(_.get(json, 'http.method') ?? '').toUpperCase(),
    url: _.get(json, urlPath[requestType], _.get(json, urlPath.default)),
    headers: /* ... */,
    auth: /* ... */,
    body: /* ... */,
    // ... 其他字段
  },
  // ...
}
```

**执行流程分析**：
1. `transformedJson.request` 对象采用统一结构定义，所有请求类型共享相同字段
2. `method` 字段在对象字面量中被**无条件声明和赋值**，没有任何条件判断跳过它
3. 对于 ws-request：
   - `requestType === 'grpc-request'` 为 `false`
   - 走 else 分支：`String(_.get(json, 'http.method') ?? '').toUpperCase()`
   - 纯 WS 请求通常没有 `http` 块，所以 `http.method` = `undefined`
   - 最终 `method` = `String(undefined ?? '').toUpperCase()` = `''`

**验证断点**：
- 输入：纯 WS 请求，无 `http` 块
- 检查：`'method' in result.request` → `true`
- 值检查：`result.request.method` → `''`

### 2.3 修正后结论

ws-request **包含** method 字段。由于所有请求类型共享同一 request 对象结构，method 字段被无条件设置，最终值为空字符串 `''`。

### 2.4 最小可复现示例

**输入 Bru 文件**：
```
meta {
  name: WS Method Test
  type: ws
}
ws {
  url: wss://api.example.com/ws
}
```

**解析命令**：
```javascript
const result = parseBruRequest(bruContent);
console.log('has method:', 'method' in result.request);
console.log('method value:', JSON.stringify(result.request.method));
console.log('request keys:', Object.keys(result.request));
```

**预期输出**：
```
has method: true
method value: ""
request keys: [ 'method', 'url', 'headers', 'auth', 'body', 'script', 'vars', 'assertions', 'tests', 'docs' ]
```

---

## 分析三：gRPC 与 WS 的 body.mode 来源

### 3.1 文档原有说法

| 请求类型 | body.mode 源字段 | 默认值 |
|----------|-----------------|--------|
| grpc-request | json.grpc.body | 'grpc' |
| ws-request | json.ws.body | 'ws' |

### 3.2 源码真实行为

**关键代码**（第 74-94 行）：
```javascript
if (requestType === 'grpc-request') {
  // gRPC 请求体
  transformedJson.request.body = _.get(json, 'body', {
    mode: 'grpc',  // ← 硬编码在默认对象中
    grpc: _.get(json, 'body.grpc', [
      { name: 'message 1', content: '{}' }
    ])
  });
} else if (requestType === 'ws-request') {
  // WebSocket 请求体
  transformedJson.request.body = _.get(json, 'body', {
    mode: 'ws',  // ← 硬编码在默认对象中
    ws: _.get(json, 'body.ws', [
      { name: 'message 1', content: '{}' }
    ])
  });
}
```

**注意**：对比 HTTP/GraphQL 的处理方式（第 95-100 行）：
```javascript
} else {
  // HTTP / GraphQL：显式读取 http.body 设置 mode
  transformedJson.request.body = _.get(json, 'body', {});
  transformedJson.request.body.mode = _.get(json, 'http.body', 'none');
}
```

**执行流程分析**：

**场景 A：Bru 文件无任何 body:* 块**
1. `_.get(json, 'body')` → `undefined`
2. 回退到第二个参数（默认对象）
3. `body = { mode: 'grpc', grpc: [...] }` 或 `{ mode: 'ws', ws: [...] }`
4. **结果**：`body.mode` = `'grpc'` 或 `'ws'`

**场景 B：Bru 文件有 body:json 块**
1. `_.get(json, 'body')` → `{ json: "{\"hello\": \"world\"}" }`
2. 直接使用该对象，不回退到默认
3. **结果**：`body = { json: "..." }` → **无 mode 字段**！

**场景 C：Bru 文件有 body:grpc 块**
1. `_.get(json, 'body')` → `{ grpc: [...] }`
2. 直接使用该对象
3. **结果**：`body = { grpc: [...] }` → **无 mode 字段**！

**关键发现**：
- `json.grpc.body` 和 `json.ws.body` **从未被读取**用于设置 body.mode
- 这两个字段仅在 `stringifyBruRequest` 函数中被**写入** Bru 文件（第 167-169 行、第 188-191 行）

### 3.3 修正后结论

| 请求类型 | body.mode 来源 | 说明 |
|----------|---------------|------|
| grpc-request | 硬编码默认值 | 来自 `_.get(json, 'body', {...})` 的第二个参数，**不读取** `json.grpc.body` |
| ws-request | 硬编码默认值 | 来自 `_.get(json, 'body', {...})` 的第二个参数，**不读取** `json.ws.body` |
| http-request | json.http.body | 显式读取 http.body 字段设置 mode |
| graphql-request | json.http.body | 显式读取 http.body 字段设置 mode |

**补充说明**：
1. 无 body 块时：使用硬编码默认对象，mode = 'grpc' 或 'ws'
2. 有 body:* 块时：直接使用该 body 对象，**可能缺少 mode 字段**（如 body:json 只有 json 字段）
3. `grpc.body` 和 `ws.body` 仅用于 **stringify 序列化**时写入 Bru 文件，解析时不读取

### 3.4 最小可复现示例

#### 示例 1：纯 gRPC 无 body 块

**输入 Bru 文件**：
```
meta {
  name: gRPC No Body
  type: grpc
}
grpc {
  url: grpc://localhost:50051
}
```

**解析命令**：
```javascript
const result = parseBruRequest(bruContent);
console.log('body.mode:', result.request.body.mode);
console.log('body keys:', Object.keys(result.request.body));
```

**预期输出**：
```
body.mode: "grpc"
body keys: [ 'mode', 'grpc' ]
```

---

#### 示例 2：gRPC + body:json 块

**输入 Bru 文件**：
```
meta {
  name: gRPC With JSON
  type: grpc
}
grpc {
  url: grpc://localhost:50051
}
body:json {
  {"hello": "world"}
}
```

**解析命令**：
```javascript
const result = parseBruRequest(bruContent);
console.log('body keys:', Object.keys(result.request.body));
console.log('has mode:', 'mode' in result.request.body);
console.log('body.json:', result.request.body.json);
```

**预期输出**：
```
body keys: [ 'json' ]
has mode: false
body.json: "{\"hello\": \"world\"}"
```

---

#### 示例 3：WS 无 body 块

**输入 Bru 文件**：
```
meta {
  name: WS No Body
  type: ws
}
ws {
  url: wss://api.example.com/ws
}
```

**解析命令**：
```javascript
const result = parseBruRequest(bruContent);
console.log('body.mode:', result.request.body.mode);
```

**预期输出**：
```
body.mode: "ws"
```

---

## 总结对照表

| 问题 | 原有说法 | 真实行为 | 修正结论 |
|------|---------|---------|---------|
| **HTTP/GraphQL method 默认值** | HTTP=GET, GraphQL=POST | `String(_.get(json, 'http.method') ?? '').toUpperCase()` → 无 method 时为 `''` | 所有非 gRPC 请求默认 method 都是空字符串 |
| **ws-request 的 method 字段** | 无 method 字段 | 所有请求类型共享统一 request 结构，method 无条件设置 | ws-request 包含 method 字段，值为空字符串 |
| **gRPC/WS body.mode 来源** | 来自 `grpc.body` / `ws.body` | mode 硬编码在 `_.get` 的默认对象中，不读取 `grpc.body`/`ws.body` | body.mode 是硬编码默认值，与 grpc.body/ws.body 无关，有 body:* 块时可能缺失 mode 字段 |

---

## 潜在问题与建议

### 问题 1：body.mode 缺失风险

当 gRPC/WS 请求带有 `body:json` 等非原生 body 块时，解析结果的 `body` 对象**缺少 mode 字段**，可能导致执行层判断逻辑出错。

**建议**：在解析后补充 mode 字段默认值：
```javascript
// 现有代码
transformedJson.request.body = _.get(json, 'body', { mode: 'grpc', ... });

// 改进后
transformedJson.request.body = _.get(json, 'body', {});
transformedJson.request.body.mode = _.get(json, 'body.mode', 'grpc');
// 或始终保证 mode 存在
if (!transformedJson.request.body.mode) {
  transformedJson.request.body.mode = 'grpc';
}
```

### 问题 2：method 空字符串处理

执行层需要兼容 method 为空字符串的情况，避免因 HTTP 方法无效导致请求发送失败。

**建议**：在执行层补充默认值：
```javascript
const method = request.method || 'GET';  // 默认回退到 GET
```

---

## 附录：源码位置索引

| 行为 | 文件 | 行号 |
|------|------|------|
| method 字段赋值 | packages/bruno-filestore/src/formats/bru/index.ts | 49-52 |
| gRPC body 处理 | packages/bruno-filestore/src/formats/bru/index.ts | 74-83 |
| WS body 处理 | packages/bruno-filestore/src/formats/bru/index.ts | 84-94 |
| HTTP body mode 显式设置 | packages/bruno-filestore/src/formats/bru/index.ts | 95-100 |
| stringify 时写 grpc.body | packages/bruno-filestore/src/formats/bru/index.ts | 167-169 |
| stringify 时写 ws.body | packages/bruno-filestore/src/formats/bru/index.ts | 188-191 |
