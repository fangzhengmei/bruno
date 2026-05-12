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
- [3. 流式响应消费](#3-流式响应消费)
  - [3.1 四种调用类型](#31-四种调用类型)
  - [3.2 事件处理机制](#32-事件处理机制)
  - [3.3 连接生命周期管理](#33-连接生命周期管理)

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

// 2. 根据调用类型执行序列化
switch (methodType) {
  case 'unary':
    // Unary: 直接传递消息对象，序列化由内部自动执行
    rpc = client.makeUnaryRequest(
      requestPath,
      method.requestSerialize,  // 序列化器
      method.responseDeserialize,
      messages[0],              // 单个消息
      metadata,
      callback
    );
    break;
  
  case 'server-streaming':
    // Server Streaming: 同 Unary，单个请求，多个响应
    rpc = client.makeServerStreamRequest(
      requestPath,
      method.requestSerialize,
      method.responseDeserialize,
      message,                  // 单个请求消息
      metadata
    );
    break;
  
  case 'client-streaming':
  case 'bidi-streaming':
    // Client/Bidi Streaming: 通过 write() 方法流式发送
    rpc = client.makeBidiStreamRequest(
      requestPath,
      method.requestSerialize,
      method.responseDeserialize,
      metadata
    );
    // 后续通过 rpc.write(message) 发送每个消息
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

## 3. 流式响应消费

### 3.1 四种调用类型

gRPC 支持四种调用模式，根据请求/响应是否流式区分：

```javascript
#getMethodType({ requestStream, responseStream }) {
  if (requestStream && responseStream) return 'bidi-streaming';    // 双向流
  if (requestStream) return 'client-streaming';                    // 客户端流
  if (responseStream) return 'server-streaming';                   // 服务端流
  return 'unary';                                                  // 一元调用
}
```

#### 各类型调用处理函数

| 调用类型 | 创建函数 | 消息发送方式 | 响应接收方式 |
|---------|---------|------------|------------|
| **Unary** | `makeUnaryRequest()` | 单次参数传入 | 回调函数接收 |
| **Server Streaming** | `makeServerStreamRequest()` | 单次参数传入 | data 事件流式接收 |
| **Client Streaming** | `makeClientStreamRequest()` | rpc.write() 流式发送 | 结束时回调接收 |
| **Bidi Streaming** | `makeBidiStreamRequest()` | rpc.write() 流式发送 | data 事件流式接收 |

### 3.2 事件处理机制

所有调用类型共享同一套事件处理机制：

```javascript
const setupGrpcEventHandlers = (callback, requestId, collectionUid, rpc, onComplete) => {
  let completed = false;
  const complete = () => {
    if (completed) return;
    completed = true;
    if (typeof onComplete === 'function') onComplete();
  };

  // 1. 状态事件 - 调用完成时触发
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

  // 2. 错误事件 - 调用失败时触发
  rpc.on('error', (error) => {
    const errorWithMetadata = {
      ...error,
      metadata: processGrpcMetadata(error.metadata.getMap())
    };
    callback('grpc:error', requestId, collectionUid, { error: errorWithMetadata });
    complete();
  });

  // 3. 数据事件 - 流式响应时多次触发
  rpc.on('data', (res) => {
    callback('grpc:response', requestId, collectionUid, { 
      error: null, 
      res 
    });
  });

  // 4. 结束事件 - 服务端流结束时触发
  rpc.on('end', (res) => {
    callback('grpc:server-end-stream', requestId, collectionUid, { res });
    complete();
  });

  // 5. 取消事件 - 调用被取消时触发
  rpc.on('cancel', (res) => {
    callback('grpc:server-cancel-stream', requestId, collectionUid, { res });
    complete();
  });

  // 6. 元数据事件 - 收到响应头时触发（在 data 之前）
  rpc.on('metadata', (metadata) => {
    const processed = processGrpcMetadata(metadata.getMap());
    callback('grpc:metadata', requestId, collectionUid, { metadata: processed });
  });
};
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
    if (value && typeof value === 'object' && value.type === 'Buffer' && Array.isArray(value.data)) {
      return { name, value: Buffer.from(value.data).toString('base64') };
    }
    return { name, value: value.toString() };
  });
};
```

### 3.3 连接生命周期管理

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

3. **流式响应消费**
   - 四种调用类型使用统一事件模型
   - 通过 data 事件流式接收响应数据
   - 完善的连接生命周期管理，防止资源泄漏

### 关键代码位置

- **gRPC 客户端实现**: `packages/bruno-requests/src/grpc/grpc-client.js`
- **gRPC 消息生成器**: `packages/bruno-requests/src/grpc/grpcMessageGenerator.js`
- **gRPC 导出入口**: `packages/bruno-requests/src/grpc/index.ts`
