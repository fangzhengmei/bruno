# gRPC 调用流程详解

本文档详细说明 gRPC 调用中 **Payload 序列化**、**Metadata 装配** 和 **流式响应消费** 三大核心流程的实现机制。

## 目录

- [整体架构](#整体架构)
- [1. Payload 序列化与反序列化](#1-payload-序列化与反序列化)
  - [1.1 Protocol Buffers 加载配置](#11-protocol-buffers-加载配置)
  - [1.2 请求序列化流程](#12-请求序列化流程)
  - [1.3 响应反序列化流程](#13-响应反序列化流程)
- [2. Metadata 处理机制](#2-metadata-处理机制)
  - [2.1 Metadata 数据结构](#21-metadata-数据结构)
  - [2.2 Metadata 装配流程](#22-metadata-装配流程)
  - [2.3 CallCredentials 与 Metadata 集成](#23-callcredentials-与-metadata-集成)
- [3. 四种调用类型对照详解](#3-四种调用类型对照详解)
  - [3.1 类型区分方式](#31-类型区分方式)
  - [3.2 各类型对照总表](#32-各类型对照总表)
  - [3.3 Unary 一元调用](#33-unary-一元调用)
  - [3.4 Server Streaming 服务端流](#34-server-streaming-服务端流)
  - [3.5 Client Streaming 客户端流](#35-client-streaming-客户端流)
  - [3.6 Bidi Streaming 双向流](#36-bidi-streaming-双向流)
  - [3.7 参数签名对比](#37-参数签名对比)
- [4. 流式响应消费](#4-流式响应消费)
  - [4.1 事件处理机制](#41-事件处理机制)
  - [4.2 连接生命周期管理](#42-连接生命周期管理)

---

## 整体架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           gRPC 调用流程                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────────────┐    │
│  │  Proto 加载  │────▶│ Metadata 装配 │────▶│  流式/非流式调用处理  │    │
│  └──────────────┘     └──────────────┘     └──────────────────────┘    │
│         │                      │                        │               │
│         ▼                      ▼                        ▼               │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────────────┐    │
│  │ requestSerialize │  │  CallCredentials  │  │  事件回调处理        │    │
│  └──────────────┘     └──────────────┘     └──────────────────────┘    │
│         │                      │                        │               │
│         ▼                      ▼                        ▼               │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────────────┐    │
│  │ responseDeserialize │ │  ChannelCredentials │ │  连接池管理         │    │
│  └──────────────┘     └──────────────┘     └──────────────────────┘    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Payload 序列化与反序列化

### 1.1 Protocol Buffers 加载配置

#### 核心配置选项

gRPC 客户端在加载 `.proto` 文件时使用以下关键配置：

```javascript
const configOptions = {
  keepCase: true,
  longs: String,      // 大整数转字符串避免精度丢失
  enums: String,      // 枚举值转字符串便于阅读
  bytes: String,      // bytes 转 base64 字符串
  defaults: true,     // 包含默认值
  oneofs: true,       // 包含 oneof 字段信息
  json: true          // 启用 JSON 格式支持
};
```

**关键设计决策：**

- `longs: String` - JavaScript 的 `Number` 类型只能安全表示到 `2^53 - 1`，gRPC 的 `int64`/`uint64` 可能超出此范围，因此转为字符串保留精度
- `bytes: String` - Buffer 类型在序列化时会转为 base64 编码的字符串，便于 JSON 传输
- `keepCase: true` - 保持字段的原始命名风格，不强制转为驼峰命名

#### Proto 加载方式

系统支持两种 Proto 加载方式：

1. **Server Reflection** - 通过 gRPC 反射服务动态获取服务定义
2. **Local Proto File** - 直接加载本地 `.proto` 文件

```javascript
// 反射方式加载
const { client, services, callOptions } = await this.#getReflectionClient(
  targetHost, 
  credentials, 
  metadata, 
  mergedChannelOptions
);

// 本地文件加载
const protoDefinition = await protoLoader.load(filePath, { 
  ...configOptions, 
  includeDirs 
});
```

### 1.2 请求序列化流程

#### 序列化函数来源

序列化函数由 `@grpc/proto-loader` 自动生成，绑定到每个方法定义：

```javascript
methods.forEach((method) => {
  this.methods.set(method.path, {
    ...method,
    requestSerialize: method.requestSerialize,  // 序列化函数
    responseDeserialize: method.responseDeserialize,  // 反序列化函数
    requestStream: method.requestStream,
    responseStream: method.responseStream,
    type: this.#getMethodType(method)
  });
});
```

#### 请求数据处理流程

```javascript
// 1. 解析用户输入的 JSON 消息
let messages = request.body.grpc;
messages = messages.map(({ content }) => safeJsonParse(content, 'message content'));

// 2. 根据调用类型分发到对应处理函数
const methodType = this.#getMethodType(method);
switch (methodType) {
  case 'unary':
    this.#handleUnaryResponse({ client, requestId, requestPath, method, messages, metadata, collectionUid });
    break;
  case 'client-streaming':
    this.#handleClientStreamingResponse({ client, requestId, requestPath, method, metadata, collectionUid });
    break;
  case 'server-streaming':
    this.#handleServerStreamingResponse({ client, requestId, requestPath, method, messages, metadata, collectionUid });
    break;
  case 'bidi-streaming':
    this.#handleBidiStreamingResponse({ client, requestId, requestPath, method, messages, metadata, collectionUid });
    break;
}
```

#### 流式发送实现

对于客户端流和双向流场景，消息通过 `write()` 方法逐个发送：

```javascript
sendMessage(requestId, collectionUid, body) {
  const entry = this.activeConnections.get(requestId);
  const rpc = entry?.rpc;

  if (rpc) {
    const parsedBody = typeof body === 'string' 
      ? safeJsonParse(body, 'request body') 
      : body;
    
    // write() 内部会自动调用 requestSerialize 进行序列化
    rpc.write(parsedBody, (error) => {
      if (error) {
        this.eventCallback('grpc:error', requestId, collectionUid, { error });
      }
    });
  }
}
```

### 1.3 响应反序列化流程

反序列化流程由 gRPC 库内部自动处理，通过 `responseDeserialize` 函数将二进制响应转为 JavaScript 对象：

```javascript
// 响应事件回调
rpc.on('data', (res) => {
  // res 已经通过 responseDeserialize 反序列化为对象
  callback('grpc:response', requestId, collectionUid, { 
    error: null, 
    res 
  });
});
```

---

## 2. Metadata 处理机制

### 2.1 Metadata 数据结构

gRPC 的 `Metadata` 类是 HTTP/2 Header 的抽象：

```javascript
import { Metadata } from '@grpc/grpc-js';

const metadata = new Metadata();

// 添加元数据
metadata.add('authorization', 'Bearer token123');
metadata.add('x-request-id', 'uuid-123');

// 获取元数据
const map = metadata.getMap();  // { authorization: 'Bearer token123', 'x-request-id': 'uuid-123' }
```

**Metadata 特性：**
- 键名自动转为小写
- 同一键可对应多个值
- 支持二进制值（键名以 `-bin` 结尾）
- 底层通过 HTTP/2 HEADERS 帧传输

### 2.2 Metadata 装配流程

#### 标准装配流程

```javascript
async startConnection({ request, ... }) {
  // 1. 创建 Metadata 实例
  const metadata = new Metadata();
  
  // 2. 将请求头逐个添加到 Metadata
  Object.entries(request.headers).forEach(([name, value]) => {
    metadata.add(name, value);
  });
  
  // 3. 传递给 gRPC 调用
  this.#handleConnection({
    client,
    metadata,  // 注入到调用
    ...
  });
}
```

#### 特殊处理：User-Agent

User-Agent 需要通过 Channel Option 特殊处理：

```javascript
// 提取 User-Agent（大小写不敏感）
const userAgentKey = Object.keys(request.headers).find(
  (key) => key.toLowerCase() === 'user-agent'
);
const userAgentValue = userAgentKey ? request.headers[userAgentKey] : null;

// 设置为 channel option，会追加到默认 User-Agent 前面
if (userAgentValue) {
  mergedChannelOptions['grpc.primary_user_agent'] = userAgentValue;
}
```

### 2.3 CallCredentials 与 Metadata 集成

对于需要通过 Credentials 传递 Metadata 的场景（如反射调用）：

```javascript
/**
 * 创建 Call Options，将 Metadata 包装为 CallCredentials
 * @param {grpc.Metadata} metadata - 要发送的元数据
 * @returns {Object} 包含 credentials 的 callOptions
 */
#createCallOptions(metadata) {
  if (metadata && Object.keys(metadata.getMap()).length > 0) {
    // 通过 Metadata Generator 创建 CallCredentials
    const callCredentials = CallCredentials.createFromMetadataGenerator(
      (options, callback) => {
        callback(null, metadata);  // 每次调用时注入 Metadata
      }
    );
    return { credentials: callCredentials };
  }
  return {};
}

// 使用场景：反射调用
const { client, services, callOptions } = await this.#getReflectionClient(...);
const methods = await client.listServices('*', callOptions);
```

**关键知识点：**
- `ChannelCredentials` - 通道级凭证（TLS/SSL），整个通道共用
- `CallCredentials` - 调用级凭证，每个 RPC 调用独立
- 通过 `createFromMetadataGenerator` 可以为每次调用动态注入 Metadata

---

## 3. 四种调用类型对照详解

### 3.1 类型区分方式

gRPC 支持四种调用模式，根据请求/响应是否流式区分：

```javascript
#getMethodType({ requestStream, responseStream }) {
  if (requestStream && responseStream) return 'bidi-streaming';    // 双向流
  if (requestStream) return 'client-streaming';                    // 客户端流
  if (responseStream) return 'server-streaming';                   // 服务端流
  return 'unary';                                                  // 一元调用
}
```

### 3.2 各类型对照总表

| 维度 | Unary (一元调用) | Server Streaming (服务端流) | Client Streaming (客户端流) | Bidi Streaming (双向流) |
|-----|----------------|--------------------------|----------------------------|------------------------|
| **处理函数入口** | `#handleUnaryResponse()` | `#handleServerStreamingResponse()` | `#handleClientStreamingResponse()` | `#handleBidiStreamingResponse()` |
| **gRPC 原生 API** | `client.makeUnaryRequest()` | `client.makeServerStreamRequest()` | `client.makeClientStreamRequest()` | `client.makeBidiStreamRequest()` |
| **请求发送入口** | 创建时直接传入 `messages[0]` | 创建时直接传入 `messages[0]` | 创建后通过 `rpc.write()` 发送 | 创建后通过 `rpc.write()` 发送 |
| **Payload 序列化触发点** | 创建调用时内部自动序列化 | 创建调用时内部自动序列化 | 每次 `rpc.write()` 时序列化 | 每次 `rpc.write()` 时序列化 |
| **Metadata 传递位置** | 第 5 个参数 (message 后) | 第 5 个参数 (message 后) | 第 4 个参数 (无 message) | 第 4 个参数 (无 message) |
| **响应回调参数** | 第 6 个参数 (有 callback) | 第 6 个参数 (有 callback) | 第 5 个参数 (有 callback) | 无 callback 参数 |
| **响应消费方式** | callback 单次回调 | `data` 事件流式多次触发 | callback 单次回调 (end 后触发) | `data` 事件流式多次触发 |
| **结束事件** | `status` 事件 | `end` + `status` | `status` 事件 | `end` + `status` |
| **是否需要 `rpc.end()`** | 否 | 否 | 是 (标记发送结束) | 是 (标记发送结束) |
| **requestStream** | `false` | `false` | `true` | `true` |
| **responseStream** | `false` | `true` | `false` | `true` |

---

### 3.3 Unary 一元调用

**定义：单次请求，单次响应

```javascript
#handleUnaryResponse({ client, requestId, requestPath, method, messages, metadata, collectionUid }) {
  const rpc = client.makeUnaryRequest(
    requestPath,                // 1: 方法路径
    method.requestSerialize, // 2: 请求序列化函数
    method.responseDeserialize, // 3: 响应反序列化函数
    messages[0],             // 4: 请求消息 (创建时传入，立即序列化)
    metadata,              // 5: Metadata
    (error, res) => {       // 6: 响应回调 (单次触发)
      this.eventCallback('grpc:response', requestId, collectionUid, { error, res });
    }
  );
  this.#addConnection(requestId, { rpc, client });
  setupGrpcEventHandlers(this.eventCallback, requestId, collectionUid, rpc, () => this.#removeConnection(requestId));
}
```

**调用链路逐项对齐：
- **请求发送入口**：第 4 个参数 `messages[0]`，调用创建时立即发送
- **Payload 序列化触发点**：调用 `makeUnaryRequest` 内部自动调用 `requestSerialize`
- **Metadata 传递位置**：第 5 个参数，在 message 之后
- **响应消费**：第 6 个参数 callback，服务端返回响应时触发一次
- **结束事件**：`status` 事件标记整个调用完成

---

### 3.4 Server Streaming 服务端流

**定义：单次请求，多次响应**

```javascript
#handleServerStreamingResponse({ client, requestId, requestPath, method, messages, metadata, collectionUid }) {
  const message = messages[0];
  const rpc = client.makeServerStreamRequest(
    requestPath,                // 1: 方法路径
    method.requestSerialize, // 2: 请求序列化函数
    method.responseDeserialize, // 3: 响应反序列化函数
    message,                 // 4: 请求消息 (创建时传入，立即序列化)
    metadata,              // 5: Metadata
    (error, res) => {       // 6: 响应回调 (流结束时触发一次)
      this.eventCallback('grpc:response', requestId, collectionUid, { error, res });
    }
  );
  this.#addConnection(requestId, { rpc, client });
  setupGrpcEventHandlers(this.eventCallback, requestId, collectionUid, rpc, () => this.#removeConnection(requestId));
}
```

**调用链路逐项对齐：
- **请求发送入口**：第 4 个参数 `message`，同 Unary，调用创建时立即发送
- **Payload 序列化触发点**：调用 `makeServerStreamRequest` 内部自动序列化
- **Metadata 传递位置**：第 5 个参数，在 message 之后
- **响应消费**：主要通过 `data` 事件流式接收，每个响应消息触发一次；callback 在流结束时触发一次
- **结束事件**：先触发 `end` 事件标记流结束，再触发 `status` 事件标记调用完成

---

### 3.5 Client Streaming 客户端流

**定义：多次请求，单次响应**

```javascript
#handleClientStreamingResponse({ client, requestId, requestPath, method, metadata, collectionUid }) {
  const rpc = client.makeClientStreamRequest(
    requestPath,                // 1: 方法路径
    method.requestSerialize, // 2: 请求序列化函数
    method.responseDeserialize, // 3: 响应反序列化函数
    metadata,              // 4: Metadata (注意：没有 message 参数！)
    (error, res) => {       // 5: 响应回调 (服务端响应时触发一次)
      this.eventCallback('grpc:response', requestId, collectionUid, { error, res });
    }
  );
  this.#addConnection(requestId, { rpc, client });
  setupGrpcEventHandlers(this.eventCallback, requestId, collectionUid, rpc, () => this.#removeConnection(requestId));
}
```

**调用链路逐项对齐：
- **请求发送入口**：创建调用时**不传入消息**，仅建立连接；后续通过 `sendMessage()` → `rpc.write(message)` 逐个发送
- **Payload 序列化触发点**：每次调用 `rpc.write()` 时，gRPC 内部自动调用 `requestSerialize` 序列化
- **Metadata 传递位置**：第 4 个参数（注意：没有 message 参数，位置前移！）
- **响应消费**：第 5 个参数 callback，服务端在收到所有请求后返回单次响应
- **结束事件**：必须调用 `rpc.end()` 标记发送完成；最终通过 `status` 事件标记调用完成

---

### 3.6 Bidi Streaming 双向流

**定义：多次请求，多次响应**

```javascript
#handleBidiStreamingResponse({ client, requestId, requestPath, method, messages, metadata, collectionUid }) {
  const rpc = client.makeBidiStreamRequest(
    requestPath,                // 1: 方法路径
    method.requestSerialize, // 2: 请求序列化函数
    method.responseDeserialize, // 3: 响应反序列化函数
    metadata               // 4: Metadata (注意：既没有 message 参数，也没有 callback 参数！)
  );
  this.#addConnection(requestId, { rpc, client });
  setupGrpcEventHandlers(this.eventCallback, requestId, collectionUid, rpc, () => this.#removeConnection(requestId));
}
```

**调用链路逐项对齐：
- **请求发送入口**：创建调用时**既不传入消息也不传入 callback**；后续通过 `sendMessage()` → `rpc.write(message)` 流式发送
- **Payload 序列化触发点**：每次调用 `rpc.write()` 时自动序列化
- **Metadata 传递位置**：第 4 个参数（无 message，无 callback）
- **响应消费**：全部通过 `data` 事件流式接收，无 callback 参数
- **结束事件**：必须调用 `rpc.end()` 标记发送完成；服务端流结束时先触发 `end` 事件，再触发 `status` 事件

---

### 3.7 参数签名对比

| 参数位置 | makeUnaryRequest | makeServerStreamRequest | makeClientStreamRequest | makeBidiStreamRequest |
|---------|----------------|------------------------|------------------------|----------------------|
| 1 | path | path | path | path |
| 2 | serialize | serialize | serialize | serialize |
| 3 | deserialize | deserialize | deserialize | deserialize |
| 4 | **message** | **message** | **metadata** | **metadata** |
| 5 | **metadata** | **metadata** | **callback** | - |
| 6 | **callback** | **callback** | - | - |

> **参数位置关键差异：**
> - **Unary/Server Stream**: 有 message 参数（第 4 位），metadata 在第 5 位，callback 在第 6 位
> - **Client Stream**: 无 message 参数，metadata 前移到第 4 位，callback 在第 5 位
> - **Bidi Stream**: 无 message 参数，无 callback 参数，metadata 在第 4 位

---

## 4. 流式响应消费

### 4.1 事件处理机制

所有调用类型共享同一套事件处理机制。**⚠️ 重要：以下所有时序描述严格基于代码证据，不做任何超出代码语义的顺序假设。**

```javascript
const setupGrpcEventHandlers = (callback, requestId, collectionUid, rpc, onComplete) => {
  // ┌─────────────────────────────────────────────────────────────────┐
  // │ 【代码可证据的保证 1】：完成状态防重入保护                          │
  // │ - completed 标志位确保只会触发一次完成动作                         │
  // │ - status / error / end / cancel 是"竞争关系"，谁先触发谁就完成       │
  // │ - 一旦其中任意一个触发，后续其他完成事件都会被忽略                  │
  // └─────────────────────────────────────────────────────────────────┘
  let completed = false;
  const complete = () => {
    if (completed) return;  // 直接丢弃后续完成事件
    completed = true;
    if (typeof onComplete === 'function') onComplete();
  };

  // ┌─────────────────────────────────────────────────────────────────┐
  // │ 【完成事件 1】：status - 调用正常/异常结束                          │
  // │ - 触发后调用 complete() 标记完成                                   │
  // │ - 包含响应状态码和响应 Metadata                                    │
  // └─────────────────────────────────────────────────────────────────┘
  rpc.on('status', (status, res) => {
    const statusWithMetadata = {
      ...status,
      metadata: processGrpcMetadata(status.metadata.getMap())
    };
    callback('grpc:status', requestId, collectionUid, { 
      status: statusWithMetadata, 
      res 
    });
    complete();
  });

  // ┌─────────────────────────────────────────────────────────────────┐
  // │ 【完成事件 2】：error - 调用发生错误                                │
  // │ - 触发后调用 complete() 标记完成                                   │
  // │ - 包含错误信息和可能的 Metadata                                     │
  // └─────────────────────────────────────────────────────────────────┘
  rpc.on('error', (error) => {
    const errorWithMetadata = {
      ...error,
      metadata: processGrpcMetadata(error.metadata.getMap())
    };
    callback('grpc:error', requestId, collectionUid, { error: errorWithMetadata });
    complete();
  });

  // ┌─────────────────────────────────────────────────────────────────┐
  // │ 【数据事件】：data - 收到响应数据                                   │
  // │ - 流式调用时可能触发 0 ~ N 次                                      │
  // │ - ❗ 关键：不触发 complete，不影响完成状态                           │
  // └─────────────────────────────────────────────────────────────────┘
  rpc.on('data', (res) => {
    callback('grpc:response', requestId, collectionUid, { 
      error: null, 
      res 
    });
  });

  // ┌─────────────────────────────────────────────────────────────────┐
  // │ 【完成事件 3】：end - 服务端流结束                                  │
  // │ - 触发后调用 complete() 标记完成                                   │
  // │ - 仅 Server Stream / Bidi Stream 可能触发                          │
  // └─────────────────────────────────────────────────────────────────┘
  rpc.on('end', (res) => {
    callback('grpc:server-end-stream', requestId, collectionUid, { res });
    complete();
  });

  // ┌─────────────────────────────────────────────────────────────────┐
  // │ 【完成事件 4】：cancel - 调用被取消                                │
  // │ - 触发后调用 complete() 标记完成                                   │
  // │ - 主动取消调用时触发（如用户点击取消按钮）                          │
  // └─────────────────────────────────────────────────────────────────┘
  rpc.on('cancel', (res) => {
    callback('grpc:server-cancel-stream', requestId, collectionUid, { res });
    complete();
  });

  // ┌─────────────────────────────────────────────────────────────────┐
  // │ 【元数据事件】：metadata - 收到响应头                              │
  // │ - ❗ 关键：不触发 complete，不影响完成状态                           │
  // └─────────────────────────────────────────────────────────────────┘
  rpc.on('metadata', (metadata) => {
    const processed = processGrpcMetadata(metadata.getMap());
    callback('grpc:metadata', requestId, collectionUid, { metadata: processed });
  });
};
```

---

### 4.1.1 完成条件矩阵

| 事件 | 是否触发 complete | 含义 | 连接是否移除 | 适用调用类型 |
|-----|-----------------|------|-------------|------------|
| **status** | ✅ 是 | gRPC 调用结束（无论成功或失败，都有状态码） | ✅ 是（通过 onComplete 回调） | 所有类型 |
| **error** | ✅ 是 | 调用发生错误（如连接失败、认证失败等） | ✅ 是 | 所有类型 |
| **end** | ✅ 是 | 服务端流正常结束（无更多数据） | ✅ 是 | Server Stream / Bidi Stream |
| **cancel** | ✅ 是 | 调用被主动取消 | ✅ 是 | 所有类型 |
| **data** | ❌ 否 | 收到一条响应数据 | ❌ 否（可继续接收） | Server Stream / Bidi Stream |
| **metadata** | ❌ 否 | 收到响应头 Metadata | ❌ 否（等待数据或结束） | 所有类型 |

> **代码证据**：`onComplete` 回调实际执行 `this.#removeConnection(requestId)`，见各 `handleXxxResponse` 函数第 2 行。

---

### 4.1.2 可能出现的事件顺序

#### 【代码可证据的保证 2】：事件类型分组
- ✅ **完成事件组**（互斥，仅一个能触发完成）：`status` / `error` / `end` / `cancel`
- ✅ **非完成事件组**（无互斥，可随时触发）：`data` / `metadata`
- ✅ **非完成事件只能在完成事件触发之前或同时触发**（完成后连接可能已关闭）

#### 【非保证】：以下为实际观察到的可能顺序，但代码不作保证
> ⚠️ 以下顺序仅为经验总结，**不代表代码契约**，实际执行顺序取决于 gRPC 底层实现和网络状况

| 调用类型 | 常见事件顺序 | 说明 |
|---------|------------|------|
| **Unary** | `metadata` → `status` | 无 data 事件，响应通过 callback 返回 |
| **Server Stream** | `metadata` → (`data` × N) → `end` → `status` | end 先触发标记完成，status 作为流的收尾 |
| **Client Stream** | `metadata` → `status` | 无 data 事件，响应通过 callback 返回，客户端 write 完成后调用 end() |
| **Bidi Stream** | `metadata` → (`data` × N，可与 write 并发) → `end` → `status` | 读写完全异步 |

#### 【代码证据】：为什么我们不做顺序保证？
```javascript
// Bruno 代码中只是单纯注册监听器，没有任何同步原语保证顺序
rpc.on('status', handler);  // 谁先到达谁先触发
rpc.on('error', handler);   // gRPC 库内部的事件触发顺序不暴露给 Bruno
rpc.on('end', handler);     // 代码只保证"第一个到达的完成事件关闭连接"
rpc.on('cancel', handler);
// ─────────────────────────────────────────────────────────────────────
// ❌ 代码中不存在：Promise.all / Promise.race / 队列 / 排序等控制机制
// ❌ 代码中不存在：metadata 事件的回调中等待 data 事件的逻辑
// ❌ 代码中不存在：end 和 status 之间的先后依赖逻辑
```

#### Metadata 后处理

响应 Metadata 需要特殊处理，包括二进制值的 base64 编码：

```javascript
const processGrpcMetadata = (metadata) => {
  return Object.entries(metadata).map(([name, value]) => {
    // 处理数组值
    if (Array.isArray(value)) {
      return {
        name,
        value: value.map((v) => {
          // 二进制 Buffer 转 base64
          if (v && typeof v === 'object' && v.type === 'Buffer' && Array.isArray(v.data)) {
            return Buffer.from(v.data).toString('base64');
          }
          return v.toString();
        }).join(', ')
      };
    }
    // 处理单个值
    if (value && typeof v === 'object' && v.type === 'Buffer' && Array.isArray(v.data)) {
      return { name, value: Buffer.from(value.data).toString('base64') };
    }
    return { name, value: value.toString() };
  });
};
```

### 4.2 连接生命周期管理

#### 连接池管理

```javascript
class GrpcClient {
  constructor(eventCallback) {
    this.activeConnections = new Map();  // 连接池：requestId -> { rpc, client }
    this.methods = new Map();             // 方法缓存：methodPath -> methodDef
    this.eventCallback = eventCallback;
  }

  // 添加连接（取消旧连接，添加新连接）
  #addConnection(requestId, { rpc, client }) {
    this.#cancelAndCloseConnection(requestId);  // 先关闭旧连接防止泄漏
    this.activeConnections.set(requestId, { rpc, client });
    
    this.eventCallback('grpc:connections-changed', {
      activeConnectionIds: this.getActiveConnectionIds()
    });
  }

  // 移除连接（关闭客户端通道）
  #removeConnection(requestId) {
    const entry = this.activeConnections.get(requestId);
    if (!entry) return;

    // 关闭客户端通道，销毁子通道，停止重连定时器
    if (entry.client && typeof entry.client.close === 'function') {
      entry.client.close();
    }
    this.activeConnections.delete(requestId);
    
    this.eventCallback('grpc:connections-changed', {
      activeConnectionIds: this.getActiveConnectionIds()
    });
  }

  // 取消并关闭连接
  #cancelAndCloseConnection(requestId) {
    const entry = this.activeConnections.get(requestId);
    if (!entry) return;

    if (entry.rpc && typeof entry.rpc.cancel === 'function') {
      entry.rpc.cancel();  // 取消正在进行的 RPC
    }
    this.#removeConnection(requestId);
  }
}
```

#### 调用结束处理

```javascript
// 仅结束客户端流发送，不关闭通道（响应可能还在继续）
end(requestId) {
  const entry = this.activeConnections.get(requestId);
  if (!entry) return;

  // 只标记客户端流结束，不关闭通道
  if (entry.rpc && typeof entry.rpc.end === 'function') {
    entry.rpc.end();
  }
}

// 强制取消并关闭
cancel(requestId) {
  this.#cancelAndCloseConnection(requestId);
}
```

**关键设计要点：**
- `rpc.end()` 仅表示客户端不再发送消息，不影响响应接收
- `rpc.cancel()` 强制终止整个调用
- `client.close()` 关闭整个通道，释放所有资源
- 必须正确处理连接关闭，防止子通道泄漏和重连定时器残留

---

## 总结

### 核心流程回顾

1. **Payload 序列化**
   - 通过 proto-loader 生成序列化/反序列化函数
   - Unary/Server Stream：单次参数传入自动序列化
   - Client/Bidi Stream：通过 write() 流式发送自动序列化

2. **Metadata 处理**
   - 标准 Headers 直接转为 Metadata
   - User-Agent 需要通过 channel option 处理
   - CallCredentials 用于需要动态注入 Metadata 的场景

3. **四种调用类型关键差异**
   - Unary：单入单出，message + callback 都有
   - Server Stream：单入多出，有 message + callback，响应走 data 事件
   - Client Stream：多入单出，无 message 参数（靠 write 发送），有 callback
   - Bidi Stream：多入多出，既无 message 也无 callback，全靠事件驱动

4. **流式响应消费**
   - 四种调用类型使用统一事件模型
   - 通过 data 事件流式接收响应数据
   - 完善的连接生命周期管理，防止资源泄漏

### 关键代码位置

- **gRPC 客户端实现**: `packages/bruno-requests/src/grpc/grpc-client.js`
- **gRPC 消息生成器**: `packages/bruno-requests/src/grpc/grpcMessageGenerator.js`
- **gRPC 导出入口**: `packages/bruno-requests/src/grpc/index.ts`
