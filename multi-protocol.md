# 多协议请求面板技术设计报告

## 概述

Bruno API 客户端通过统一的请求面板架构支持 REST(HTTP)、GraphQL、WebSocket 和 gRPC 四种协议。本报告详细说明如何将传输层差异收敛到统一的请求生命周期中，实现一致的用户体验。

---

## 一、协议选择入口

### 1.1 请求类型枚举

系统通过 `type` 字段区分四种协议类型：

```javascript
// packages/bruno-schema/src/collections/index.js:621
type: Yup.string().oneOf([
  'http-request',      // REST/HTTP 请求
  'graphql-request',   // GraphQL 请求
  'grpc-request',      // gRPC 请求
  'ws-request'         // WebSocket 请求
])
```

### 1.2 协议面板路由机制

请求面板根据 `type` 动态加载对应的协议特定组件：

```
HttpRequestPane        (packages/bruno-app/src/components/RequestPane/HttpRequestPane/)
  ├── QueryParams          (查询参数)
  ├── RequestHeaders       (请求头)
  ├── RequestBody          (请求体：json/text/xml/form/multipart/file/graphql)
  ├── Auth                 (认证：basic/bearer/oauth1/oauth2/digest/ntlm/wsse/apikey/awsv4)
  ├── Vars                 (环境变量)
  ├── Script               (前置/后置脚本)
  ├── Assertions           (断言)
  ├── Tests                (测试脚本)
  └── Settings             (请求设置)

GraphQLRequestPane     (packages/bruno-app/src/components/RequestPane/GraphQLRequestPane/)
  ├── QueryEditor          (GraphQL 查询编辑器)
  ├── QueryBuilder         (可视化查询构建器)
  ├── GraphQLVariables     (GraphQL 变量)
  ├── RequestHeaders       (请求头)
  └── Auth                 (认证)

GrpcRequestPane        (packages/bruno-app/src/components/RequestPane/GrpcRequestPane/)
  ├── GrpcQueryUrl         (服务地址、proto 文件、方法选择)
  ├── GrpcBody             (gRPC 消息体)
  ├── RequestHeaders       (元数据)
  └── GrpcAuth             (gRPC 认证)

WSRequestPane          (packages/bruno-app/src/components/RequestPane/WSRequestPane/)
  ├── WsQueryUrl           (WebSocket 连接地址)
  ├── WsBody               (消息发送面板)
  ├── WSSettingsPane       (连接设置)
  └── WSAuth               (WebSocket 认证)
```

### 1.3 统一发送按钮

所有协议共享相同的发送按钮组件，通过状态管理实现一致的交互体验：

```javascript
// packages/bruno-app/src/components/RequestPane/SendButton/index.js
const SendButton = ({ isLoading = false, onSend, onCancel, testId }) => {
  return (
    <Button
      variant={isLoading ? 'outline' : 'filled'}
      onClick={isLoading ? onCancel : onSend}
    >
      {isLoading ? 'Cancel' : 'Send'}
    </Button>
  );
};
```

---

## 二、传输层封装

### 2.1 请求准备阶段（Prepare）

#### 2.1.1 HTTP/GraphQL 请求准备

```javascript
// packages/bruno-electron/src/ipc/network/prepare-request.js
const prepareRequest = async (item, collection, abortController) => {
  const request = item.draft ? item.draft.request : item.request;
  
  // 1. 合并集合/文件夹级别的 headers、scripts、vars、auth
  mergeHeaders(collection, request, requestTreePath);
  mergeScripts(collection, request, requestTreePath);
  mergeVars(collection, request, requestTreePath);
  mergeAuth(collection, request, requestTreePath);

  // 2. 构建 axios 请求配置
  let axiosRequest = {
    mode: request.body.mode,
    method: request.method,
    url: request.url,
    headers,
    pathParams,
    settings,
    responseType: 'arraybuffer'
  };

  // 3. 设置认证头（支持 9 种认证方式）
  axiosRequest = setAuthHeaders(axiosRequest, request, collectionRoot);

  // 4. 根据 body mode 处理请求体
  // - json/text/xml/sparql: 直接赋值
  // - formUrlEncoded/multipartForm: 处理表单参数
  // - file: 支持流式大文件上传（20MB 阈值）
  // - graphql: 包装 query + variables

  return axiosRequest;
};
```

#### 2.1.2 gRPC 请求准备

```javascript
// packages/bruno-electron/src/ipc/network/prepare-grpc-request.js
const prepareGrpcRequest = (item, collection) => {
  const request = item.draft ? item.draft.request : item.request;
  
  return {
    url: request.url,                     // gRPC 服务地址
    service: request.service,             // 服务名
    method: request.methodName,           // 方法名
    methodType: request.methodType,       // unary/client-streaming/server-streaming/bidi-streaming
    protoDefinition: request.protoDefinition, // proto 文件内容
    metadata: request.headers,            // 元数据（HTTP/2 headers）
    messages: request.body.grpc,          // 请求消息列表
    auth: request.auth                    // 认证配置
  };
};
```

#### 2.1.3 WebSocket 请求准备

```javascript
// packages/bruno-electron/src/ipc/network/prepare-ws-request.js
const prepareWSRequest = (item, collection) => {
  const request = item.draft ? item.draft.request : item.request;
  
  return {
    url: request.url,                     // ws:// 或 wss:// 地址
    protocols: request.protocols,         // 子协议
    headers: request.headers,             // 握手时的 HTTP 头
    auth: request.auth,                   // 认证
    messages: request.body.ws             // 预定义消息列表
  };
};
```

### 2.2 传输执行器（Executor）

#### 2.2.1 统一发送入口

```javascript
// packages/bruno-app/src/providers/ReduxStore/slices/collections/actions.js
export const sendRequest = (item, collectionUid) => async (dispatch, getState) => {
  const { type } = item;
  
  // 根据请求类型分发到不同的传输执行器
  switch (type) {
    case 'http-request':
    case 'graphql-request':
      return executeHTTPRequest(item, collectionUid);
    case 'grpc-request':
      return executeGrpcRequest(item, collectionUid);
    case 'ws-request':
      return executeWebSocketRequest(item, collectionUid);
  }
};
```

#### 2.2.2 HTTP/GraphQL 执行器

```javascript
// packages/bruno-requests/src/network/index.js
import axios from 'axios';
import http from 'http';
import https from 'https';
import { HttpProxyAgent } from 'http-proxy-agent';
import { HttpsProxyAgent } from 'https-proxy-agent';
import { SocksProxyAgent } from 'socks-proxy-agent';

export const makeAxiosInstance = (config) => {
  // 支持多种代理协议：HTTP/HTTPS/SOCKS4/SOCKS5
  // 支持自定义 CA 证书
  // 支持 Keep-Alive 连接复用
  // 支持超时和重定向配置
  
  const httpAgent = new http.Agent({ keepAlive: true });
  const httpsAgent = new https.Agent({ 
    keepAlive: true,
    ca: customCerts,
    rejectUnauthorized: strictSSL
  });

  return axios.create({
    httpAgent,
    httpsAgent,
    proxy: false,  // 使用自定义代理 agent
    timeout: config.settings?.requestTimeout || 0,
    maxRedirects: config.settings?.maxRedirects ?? 5
  });
};
```

#### 2.2.3 gRPC 执行器

```javascript
// packages/bruno-requests/src/grpc/index.js
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';

export class GrpcClient {
  // 支持四种调用模式
  async unaryCall(service, method, message, metadata) {}      // 一元调用
  async clientStreaming(service, method, messages, metadata) {} // 客户端流
  async serverStreaming(service, method, message, metadata) {} // 服务端流
  async bidiStreaming(service, method, messages, metadata) {}  // 双向流
  
  // 支持 TLS/SSL 加密连接
  // 支持 Token/Basic/SSL 等认证方式
}
```

#### 2.2.4 WebSocket 执行器

```javascript
// packages/bruno-requests/src/ws/ws-client.js
import WebSocket from 'ws';

export class WsClient {
  connect(url, options) {
    // 支持自定义 headers
    // 支持 Origin/User-Agent 等握手配置
    // 支持重连策略
    // 支持心跳检测
  }
  
  send(message) {}          // 发送文本消息
  sendBinary(data) {}       // 发送二进制消息
  close() {}                // 关闭连接
  
  onOpen(callback) {}       // 连接建立事件
  onMessage(callback) {}    // 消息接收事件
  onError(callback) {}      // 错误事件
  onClose(callback) {}      // 连接关闭事件
}
```

### 2.3 脚本执行生命周期

所有协议共享相同的脚本执行钩子：

```
            ┌─────────────────────────────────────────┐
            │              请求生命周期               │
            └─────────────────────────────────────────┘

  ┌───────────┐    ┌───────────┐    ┌───────────┐    ┌───────────┐
  │  Pre-     │    │           │    │  Post-    │    │   Test    │
  │ Request   │───▶│  实际请求  │───▶│ Response  │───▶│  Script   │
  │  Script   │    │           │    │  Script   │    │           │
  └───────────┘    └───────────┘    └───────────┘    └───────────┘

  - 变量插值                  - 响应断言
  - 动态修改请求              - 测试结果收集
  - 环境变量设置              - 报告生成
```

---

## 三、响应展示与错误归一

### 3.1 统一响应状态模型

所有协议的响应都映射到相同的数据结构：

```typescript
interface UnifiedResponse {
  // 基础信息
  protocol: 'http' | 'grpc' | 'websocket';
  statusCode: number;           // HTTP 状态码 / gRPC 状态码 / WS 关闭码
  statusText: string;           // 状态描述
  
  // 时间指标
  startTime: number;
  endTime: number;
  duration: number;             // ms
  
  // 数据负载
  headers: Record<string, string>;  // HTTP headers / gRPC metadata
  body: any;                       // 响应体 / 消息列表
  trailers?: Record<string, string>; // gRPC trailers
  
  // 元数据
  size: number;                   // 字节数
  timeline: TimelineEvent[];      // 请求时间线
  
  // 错误信息（统一格式）
  error?: {
    type: 'network' | 'timeout' | 'protocol' | 'auth' | 'parse';
    message: string;
    stack?: string;
    details?: any;
  };
  
  // 执行结果
  testResults: TestResult[];
  assertionResults: AssertionResult[];
  scriptErrors: ScriptError[];
  setEnvVars: Record<string, string>;
}
```

### 3.2 响应面板组件架构

#### 3.2.1 统一响应布局容器

```
ResponsePane (统一容器)
  ├── StatusCode              (状态码展示，支持 HTTP/gRPC/WS)
  ├── ResponseTime            (响应时间)
  ├── ResponseSize            (响应大小)
  ├── ResponseStopWatch       (实时计时器，用于流式请求)
  ├── ResponsePaneActions     (操作按钮：复制/下载/书签/清除)
  │
  ├── QueryResult             (响应体展示，支持多格式预览)
  │   ├── JsonPreview
  │   ├── XmlPreview
  │   ├── HtmlPreview
  │   ├── TextPreview
  │   └── VideoPreview
  │
  ├── ResponseHeaders         (响应头 / gRPC metadata)
  ├── ResponseTrailers        (gRPC trailers，仅 gRPC 显示)
  │
  ├── TestResults             (测试结果)
  ├── ScriptError             (脚本错误)
  │
  ├── Timeline                (请求时间线)
  │   ├── TimelineItem        (通用时间线项)
  │   └── GrpcTimelineItem    (gRPC 流专用时间线)
  │
  ├── GrpcResponsePane        (gRPC 专用子面板)
  └── WsResponsePane          (WebSocket 专用子面板)
       ├── WSMessagesList          (消息列表)
       ├── WSResponseHeaders       (握手响应头)
       └── WSStatusCode            (连接状态)
```

#### 3.2.2 状态码统一映射

```javascript
// packages/bruno-app/src/components/ResponsePane/StatusCode/index.js
// packages/bruno-app/src/components/ResponsePane/GrpcResponsePane/GrpcStatusCode/index.js
// packages/bruno-app/src/components/ResponsePane/WsResponsePane/WSStatusCode/index.js

// 状态码语义统一映射
const statusCodeTypeMap = {
  // HTTP 2xx → 成功
  200: 'success', 201: 'success', 204: 'success',
  
  // HTTP 3xx → 重定向
  301: 'redirect', 302: 'redirect', 304: 'redirect',
  
  // HTTP 4xx → 客户端错误
  400: 'error', 401: 'auth-error', 403: 'error', 404: 'error',
  
  // HTTP 5xx → 服务端错误
  500: 'error', 502: 'error', 503: 'error',
  
  // gRPC 状态码映射
  0: 'success',    // OK
  1: 'error',      // CANCELLED
  2: 'error',      // UNKNOWN
  3: 'error',      // INVALID_ARGUMENT
  4: 'error',      // DEADLINE_EXCEEDED
  7: 'auth-error', // PERMISSION_DENIED
  16: 'auth-error',// UNAUTHENTICATED
  
  // WebSocket 关闭码映射
  1000: 'success',   // Normal Closure
  1001: 'info',      // Going Away
  1006: 'error',     // Abnormal Closure
  1008: 'auth-error' // Policy Violation
};
```

### 3.3 错误归一化处理

#### 3.3.1 错误类型统一分类

```javascript
// packages/bruno-electron/src/ipc/network/execute-request-error-handler.js

const errorHandlers = {
  // 网络层错误
  ENOTFOUND: () => ({
    type: 'network',
    message: 'Could not resolve host',
    userMessage: '无法连接到服务器，请检查网络连接或 URL 是否正确'
  }),
  
  ECONNREFUSED: () => ({
    type: 'network',
    message: 'Connection refused',
    userMessage: '连接被拒绝，请检查服务是否启动'
  }),
  
  ETIMEDOUT: () => ({
    type: 'timeout',
    message: 'Request timed out',
    userMessage: '请求超时，请检查网络或增加超时时间'
  }),
  
  // TLS/SSL 错误
  DEPTH_ZERO_SELF_SIGNED_CERT: () => ({
    type: 'auth',
    message: 'Self signed certificate',
    userMessage: '自签名证书不受信任，可在设置中关闭 SSL 验证'
  }),
  
  // gRPC 特有错误
  UNAVAILABLE: () => ({
    type: 'network',
    message: 'gRPC service unavailable',
    userMessage: 'gRPC 服务不可用，请检查服务地址和端口'
  }),
  
  // WebSocket 特有错误
  WS_PROTOCOL_ERROR: () => ({
    type: 'protocol',
    message: 'WebSocket protocol error',
    userMessage: 'WebSocket 协议错误，请检查服务支持的子协议'
  })
};
```

#### 3.3.2 统一错误展示组件

```javascript
// packages/bruno-app/src/components/ResponsePane/NetworkError/index.js
const NetworkError = ({ error }) => {
  // 无论底层协议是什么，都以统一的格式展示
  return (
    <div className="error-container">
      <IconAlertCircle size={20} />
      <div className="error-content">
        <h4>{error.userMessage || error.message}</h4>
        {error.details && (
          <pre className="error-details">{error.details}</pre>
        )}
        {error.stack && process.env.NODE_ENV === 'development' && (
          <details>
            <summary>调试信息</summary>
            <pre>{error.stack}</pre>
          </details>
        )}
      </div>
    </div>
  );
};
```

### 3.4 流式响应统一处理

WebSocket 和 gRPC 流式响应采用统一的时间线模式展示：

```javascript
// packages/bruno-app/src/components/ResponsePane/Timeline/index.js
const Timeline = ({ events, protocol }) => {
  // 事件按时间排序，无论来自 WebSocket 还是 gRPC 流
  const sortedEvents = [...events].sort((a, b) => a.timestamp - b.timestamp);
  
  return (
    <div className="timeline">
      {sortedEvents.map((event, index) => (
        protocol === 'grpc' 
          ? <GrpcTimelineItem key={index} event={event} />
          : <TimelineItem key={index} event={event} />
      ))}
    </div>
  );
};

// 事件统一结构
interface TimelineEvent {
  id: string;
  type: 'send' | 'receive' | 'connect' | 'disconnect' | 'metadata' | 'status';
  timestamp: number;
  direction: 'in' | 'out';
  data: any;
  contentType?: string;
  size?: number;
}
```

---

## 四、架构总结

### 4.1 分层架构图

```
┌─────────────────────────────────────────────────────────────┐
│                        UI 表现层                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  HttpRequest │  │  GraphQLReq  │  │   GrpcReq    │       │
│  │     Pane     │  │     Pane     │  │     Pane     │  ...  │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│         │                 │                 │                 │
│         └─────────────────┴─────────────────┘                 │
│                           │                                   │
│                    ┌───────────┐                              │
│                    │ SendButton│                              │
│                    └───────────┘                              │
└───────────────────────────┬───────────────────────────────────┘
                            │
┌───────────────────────────▼───────────────────────────────────┐
│                      状态管理层 (Redux)                        │
│  ┌────────────────────────────────────────────────────────┐  │
│  │                   sendRequest action                    │  │
│  └────────────────────────────────────────────────────────┘  │
└───────────────────────────┬───────────────────────────────────┘
                            │
┌───────────────────────────▼───────────────────────────────────┐
│                      请求准备层 (Prepare)                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ prepareHTTP  │  │ prepareGrpc  │  │  prepareWS   │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
└───────────────────────────┬───────────────────────────────────┘
                            │
┌───────────────────────────▼───────────────────────────────────┐
│                      传输执行层 (Executor)                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  Axios/HTTP  │  │  gRPC-JS     │  │     WS       │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
└───────────────────────────┬───────────────────────────────────┘
                            │
┌───────────────────────────▼───────────────────────────────────┐
│                      响应归一化层 (Normalize)                    │
│  ┌────────────────────────────────────────────────────────┐  │
│  │                UnifiedResponse 转换器                   │  │
│  └────────────────────────────────────────────────────────┘  │
└───────────────────────────┬───────────────────────────────────┘
                            │
┌───────────────────────────▼───────────────────────────────────┐
│                      响应展示层 (Render)                        │
│  ┌────────────────────────────────────────────────────────┐  │
│  │                  ResponsePane 容器                      │  │
│  └────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
```

### 4.2 关键设计原则

| 原则 | 说明 |
|------|------|
| **接口一致性** | 所有协议面板采用相同的 Tab 布局、发送按钮、响应展示结构 |
| **职责分离** | UI 层只负责交互，传输层只负责协议通信，数据层统一格式 |
| **可扩展性** | 新增协议只需实现 Prepare + Executor + 特定面板，无需修改核心流程 |
| **错误统一** | 所有协议错误映射到统一的错误类型和用户提示 |
| **状态共享** | 环境变量、认证配置、脚本引擎在所有协议间共享 |

### 4.3 未来扩展点

1. **SSE (Server-Sent Events)** - 可复用 WebSocket 流式面板
2. **MQTT/AMQP** - 消息队列协议，复用 Timeline 时间线组件
3. **GraphQL over WebSocket** - 订阅模式支持
4. **Dubbo/Thrift** - 其他 RPC 框架适配

---

*报告生成时间：2026-05-12*  
*基于 Bruno v118 代码库分析*
