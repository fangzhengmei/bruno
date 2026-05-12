# 多协议请求面板技术设计报告

## 概述

Bruno API 客户端通过统一的架构设计，支持 REST(HTTP)、GraphQL、WebSocket 和 gRPC 四种协议。本报告基于实际代码实现，详细说明协议选择入口、传输层封装、响应展示与错误归一三个核心部分，并标注各环节对应的文件位置和调用关系。

---

## 一、协议选择入口

### 1.1 请求类型定义

系统通过 `type` 字段区分四种协议类型，定义在 Collection Schema 中：

```javascript
// 文件: packages/bruno-schema/src/collections/index.js:621
type: Yup.string().oneOf([
  'http-request',      // REST/HTTP 请求
  'graphql-request',   // GraphQL 请求
  'grpc-request',      // gRPC 请求
  'ws-request'         // WebSocket 请求
])
```

### 1.2 统一入口面板：RequestTabPanel

所有协议共享同一个请求面板容器，通过 `item.type` 动态加载特定协议的 UI 组件：

```javascript
// 文件: packages/bruno-app/src/components/RequestTabPanel/index.js
const RequestTabPanel = () => {
  // 第 358-359 行：协议类型识别
  const isGrpcRequest = item?.type === 'grpc-request';
  const isWsRequest = item?.type === 'ws-request';

  // 第 452-460 行：渲染 URL 栏（含发送触发）
  const renderQueryUrl = () => {
    if (isGrpcRequest) {
      return <GrpcQueryUrl item={item} collection={collection} handleRun={handleRun} />;
    }
    if (isWsRequest) {
      return <WsQueryUrl item={item} collection={collection} handleRun={handleRun} />;
    }
    return <QueryUrl item={item} collection={collection} handleRun={handleRun} />;
  };

  // 第 462-483 行：渲染请求配置面板
  const renderRequestPane = () => {
    switch (item.type) {
      case 'graphql-request':
        return <GraphQLRequestPane ... />;
      case 'http-request':
        return <HttpRequestPane item={item} collection={collection} />;
      case 'grpc-request':
        return <GrpcRequestPane item={item} collection={collection} handleRun={handleRun} />;
      case 'ws-request':
        return <WSRequestPane item={item} collection={collection} handleRun={handleRun} />;
      default:
        return null;
    }
  };

  // 第 485-494 行：渲染响应展示面板
  const renderResponsePane = () => {
    switch (item.type) {
      case 'grpc-request':
        return <GrpcResponsePane item={item} collection={collection} response={item.response} />;
      case 'ws-request':
        return <WSResponsePane item={item} collection={collection} response={item.response} />;
      default:
        return <ResponsePane item={item} collection={collection} response={item.response} />;
    }
  };
};
```

**调用关系**：
- `QueryUrl`（HTTP/GraphQL）→ `packages/bruno-app/src/components/RequestPane/QueryUrl/index.js`
- `GrpcQueryUrl` → `packages/bruno-app/src/components/RequestPane/GrpcQueryUrl/index.js`
- `WsQueryUrl` → `packages/bruno-app/src/components/RequestPane/WsQueryUrl/index.js`
- `HttpRequestPane` → `packages/bruno-app/src/components/RequestPane/HttpRequestPane/index.js`
- `GraphQLRequestPane` → `packages/bruno-app/src/components/RequestPane/GraphQLRequestPane/index.js`
- `GrpcRequestPane` → `packages/bruno-app/src/components/RequestPane/GrpcRequestPane/index.js`
- `WSRequestPane` → `packages/bruno-app/src/components/RequestPane/WSRequestPane/index.js`

### 1.3 统一发送触发：handleRun

所有协议的发送动作都通过同一个 `handleRun` 函数触发：

```javascript
// 文件: packages/bruno-app/src/components/RequestTabPanel/index.js:428-451
const handleRun = async () => {
  const request = item.draft ? item.draft.request : item.request;

  // 协议特定的前置校验
  if (isGrpcRequest && !request.url) {
    toast.error('Please enter a valid gRPC server URL');
    return;
  }
  if (isGrpcRequest && !request.method) {
    toast.error('Please select a gRPC method');
    return;
  }
  if (isWsRequest && !request.url) {
    toast.error('Please enter a valid WebSocket URL');
    return;
  }

  // 统一发送入口 → 进入传输层
  if (item.requestState !== 'sending' && item.requestState !== 'queued') {
    dispatch(sendRequest(item, collection.uid)).catch((err) =>
      toast.custom((t) => <NetworkError onClose={() => toast.dismiss(t.id)} />, {
        duration: 5000
      }));
  }
};
```

**快捷键绑定**：第 68-73 行，所有协议共享 `sendRequest` 快捷键（Cmd/Ctrl + Enter）

---

## 二、传输层封装

### 2.1 发送动作分发器：sendRequest Action

`sendRequest` 是所有协议请求的统一入口，根据 `item.type` 分发到不同的传输层：

```javascript
// 文件: packages/bruno-app/src/providers/ReduxStore/slices/collections/actions.js:528-648
export const sendRequest = (item, collectionUid) => (dispatch, getState) => {
  // 第 576-577 行：协议类型识别
  const isGrpcRequest = itemCopy.type === 'grpc-request';
  const isWsRequest = itemCopy.type === 'ws-request';

  if (isGrpcRequest) {
    // 第 578-583 行：gRPC 分支 → gRPC 传输层
    sendGrpcRequest(itemCopy, collectionCopy, environment, collectionCopy.runtimeVariables)
      .then(resolve)
      .catch((err) => { toast.error(err.message); });
  } else if (isWsRequest) {
    // 第 584-589 行：WebSocket 分支 → WebSocket 传输层
    sendWsRequest(itemCopy, collectionCopy, environment, collectionCopy.runtimeVariables)
      .then(resolve)
      .catch((err) => { toast.error(err.message); });
  } else {
    // 第 590-646 行：HTTP/GraphQL 分支 → HTTP 传输层
    sendNetworkRequest(itemCopy, collectionCopy, environment, collectionCopy.runtimeVariables)
      .then((response) => {
        const { requestSent, ...responseData } = response;
        return dispatch(
          responseReceived({
            itemUid,
            collectionUid,
            response: serializedResponse,
            requestSent
          })
        );
      })
      .catch((err) => {
        // 错误归一化处理
        const request = itemCopy.draft?.request || itemCopy.request;
        const requestSent = request ? { url: request.url, method: request.method } : undefined;

        // 取消请求的特殊处理
        if (err && err.message === 'Error invoking remote method \'send-http-request\': Error: Request cancelled') {
          dispatch(responseReceived({ itemUid, collectionUid, response: null, requestSent }));
          return;
        }

        // 统一错误响应结构
        const errorResponse = {
          status: 'Error',
          isError: true,
          error: err.message ?? 'Something went wrong',
          size: 0,
          duration: 0
        };
        dispatch(responseReceived({ itemUid, collectionUid, response: errorResponse, requestSent }));
      });
  }
};
```

**调用关系**：
- `sendNetworkRequest` → `packages/bruno-app/src/utils/network/index.js:1-29`
- `sendGrpcRequest` → `packages/bruno-app/src/utils/network/index.js:31-44`
- `sendWsRequest` → `packages/bruno-app/src/utils/network/index.js:227-245`

### 2.2 HTTP/GraphQL 传输链路

#### 2.2.1 网络请求工具层

```javascript
// 文件: packages/bruno-app/src/utils/network/index.js:1-29
export const sendNetworkRequest = async (item, collection, environment, runtimeVariables) => {
  return new Promise((resolve, reject) => {
    if (['http-request', 'graphql-request'].includes(item.type)) {
      sendHttpRequest(item, collection, environment, runtimeVariables)
        .then((response) => {
          // 响应结构归一化
          if (response?.error) {
            resolve(response);
          }
          resolve({
            state: 'success',
            data: response.data,
            dataBuffer: response.dataBuffer,
            headers: response.headers,
            size: response.size,
            status: response.status,
            statusText: response.statusText,
            duration: response.duration,
            timeline: response.timeline,
            stream: response.stream,
            requestSent: response.requestSent
          });
        })
        .catch((err) => reject(err));
    }
  });
};

// 第 46-55 行：IPC 调用主进程
const sendHttpRequest = async (item, collection, environment, runtimeVariables) => {
  return new Promise((resolve, reject) => {
    const { ipcRenderer } = window;
    ipcRenderer.invoke('send-http-request', item, collection, environment, runtimeVariables)
      .then(resolve)
      .catch(reject);
  });
};
```

#### 2.2.2 主进程请求处理

```javascript
// 文件: packages/bruno-electron/src/ipc/network/index.js:1205-1243
ipcMain.handle('send-http-request', async (event, item, collection, environment, runtimeVariables) => {
  // 第 737-940 行：调用 runRequest 执行完整请求生命周期
  const response = await runRequest({ item, collection, envVars, processEnvVars, runtimeVariables, runInBackground: false });

  // 流式响应处理
  if (response.stream) {
    stream.on('data', (newData) => {
      mainWindow.webContents.send('main:http-stream-new-data', {
        collectionUid, itemUid: item.uid, seq, timestamp: Date.now(), data: parsed
      });
    });
    stream.on('close', () => {
      mainWindow.webContents.send('main:http-stream-end', { ... });
    });
  }
  return response;
});
```

#### 2.2.3 runRequest：完整请求生命周期 + configureRequest

```javascript
// 文件: packages/bruno-electron/src/ipc/network/index.js:737-940
const runRequest = async ({ item, collection, envVars, processEnvVars, runtimeVariables, runInBackground }) => {
  // 第 768 行：请求准备
  const request = await prepareRequest(item, collection, abortController);

  // 第 798-844 行：Pre-request 脚本执行
  let preRequestScriptResult = null;
  try {
    preRequestScriptResult = await runPreRequest(...);
  } catch (error) {
    preRequestError = error;
  }

  // 第 845-854 行：configureRequest - HTTP 版本（在同一文件内定义，第 101 行）
  // 功能：代理配置、SSL/TLS 证书、OAuth1/OAuth2 认证、NTLM、Axios 实例配置
  const axiosInstance = await configureRequest(
    collectionUid,
    collection,
    request,
    envVars,
    runtimeVariables,
    processEnvVars,
    collectionPath,
    collection.globalEnvironmentVariables
  );

  // 第 901-939 行：实际发送请求
  response = await axiosInstance(request);

  // Post-response 脚本执行和测试运行...

  return response;
};
```

**关键函数位置**：
- `prepareRequest` → `packages/bruno-electron/src/ipc/network/prepare-request.js`
- `configureRequest` (HTTP 版本) → `packages/bruno-electron/src/ipc/network/index.js:101`
- `makeAxiosInstance` → `packages/bruno-electron/src/ipc/network/axios-instance.js`
- `axios-instance` 源实现 → `packages/bruno-requests/src/network/axios-instance.ts`

### 2.3 gRPC 传输链路

#### 2.3.1 连接启动

```javascript
// 文件: packages/bruno-app/src/utils/network/index.js:31-44
export const sendGrpcRequest = async (item, collection, environment, runtimeVariables) => {
  return new Promise((resolve, reject) => {
    startGrpcRequest(item, collection, environment, runtimeVariables)
      .then((initialState) => {
        // 返回初始状态，实际响应通过事件监听异步更新
        resolve({ ...initialState, timeline: [] });
      })
      .catch((err) => reject(err));
  });
};

// 第 80-98 行：IPC 调用主进程启动 gRPC 连接
export const startGrpcRequest = async (item, collection, environment, runtimeVariables) => {
  return new Promise((resolve, reject) => {
    const { ipcRenderer } = window;
    const request = item.draft ? item.draft : item;
    ipcRenderer.invoke('grpc:start-connection', { request, collection, environment, runtimeVariables })
      .then(() => resolve())
      .catch((err) => reject(err));
  });
};
```

#### 2.3.2 主进程 gRPC 连接处理 + 客户端实现

```javascript
// 文件: packages/bruno-electron/src/ipc/network/grpc-event-handlers.js:155-259
ipcMain.handle('grpc:start-connection', async (event, { request, collection, environment, runtimeVariables }) => {
  // 第 158 行：请求准备
  const preparedRequest = await prepareGrpcRequest(requestCopy, collection, environment, runtimeVariables, {});

  // 第 166-186 行：获取证书和代理配置 + 调用 configureRequest (gRPC 专用版本)
  const certsAndProxyConfig = await getCertsAndProxyConfig(...);
  await configureRequest(
    preparedRequest,
    requestCopy,
    collection,
    preparedRequest.envVars,
    runtimeVariables,
    preparedRequest.processEnvVars,
    preparedRequest.promptVariables,
    certsAndProxyConfig
  );

  // 第 223-234 行：启动 gRPC 连接
  // grpcClient 来自 @usebruno/requests 包，源文件见下文
  await grpcClient.startConnection({
    request: preparedRequest,
    collection,
    rootCertificate, privateKey, certificateChain, passphrase, pfx,
    verifyOptions, includeDirs,
    proxyConfig: grpcProxyConfig
  });

  // 第 236 行：发送请求事件到时间线
  sendEvent('grpc:request', preparedRequest.uid, collection.uid, requestSent);
  return { success: true };
});
```

**gRPC 客户端实现位置**（关键修正）：
- `GrpcClient` 导出 → `packages/bruno-requests/src/index.ts:2`
- `GrpcClient` 源实现 → `packages/bruno-requests/src/grpc/grpc-client.js`
- `GrpcClient` 入口模块 → `packages/bruno-requests/src/grpc/index.ts`
- `prepareGrpcRequest` → `packages/bruno-electron/src/ipc/network/prepare-grpc-request.js`
- `configureRequest` (gRPC 版本，OAuth2 处理) → `packages/bruno-electron/src/ipc/network/prepare-grpc-request.js:32`

### 2.4 WebSocket 传输链路

#### 2.4.1 连接与发送

```javascript
// 文件: packages/bruno-app/src/utils/network/index.js:227-245
export const sendWsRequest = async (item, collection, environment, runtimeVariables) => {
  const ensureConnection = async () => {
    const connectionStatus = await isWsConnectionActive(item.uid);
    if (!connectionStatus.isActive) {
      await connectWS(item, collection, environment, runtimeVariables, { connectOnly: true });
    }
  };
  await ensureConnection();
  // 队列化消息发送（支持变量插值）
  const result = await queueWsMessage(item, collection, environment, runtimeVariables, null);
  if (result.success) { return {}; }
  else { throw new Error(result.error || 'Failed to queue messages'); }
};

// 第 212-225 行：WebSocket 连接入口
export const connectWS = async (item, collection, environment, runtimeVariables, options) => {
  return new Promise((resolve, reject) => {
    startWsConnection(item, collection, environment, runtimeVariables, options)
      .then((initialState) => resolve({ ...initialState, timeline: [] }))
      .catch((err) => reject(err));
  });
};
```

#### 2.4.2 主进程 WebSocket 处理 + 客户端实现

```javascript
// 文件: packages/bruno-electron/src/ipc/network/ws-event-handlers.js:303-388
ipcMain.handle(
  'renderer:ws:start-connection',
  async (event, { request, collection, environment, runtimeVariables, settings, options = {} }) => {
    // 第 27-215 行：prepareWsRequest - WebSocket 请求预处理（在同一文件内定义）
    // 功能：合并 headers/scripts/vars/auth，设置认证头，OAuth2 token 处理，子协议配置
    const preparedRequest = await prepareWsRequest(requestCopy, collection, environment, runtimeVariables, {});

    // 第 318-325 行：自动发送预定义消息
    if (!connectOnly) {
      const hasMessages = preparedRequest.body.ws.some((msg) => msg.content.length);
      if (hasMessages) {
        preparedRequest.body.ws.forEach((message) => {
          wsClient.queueMessage(preparedRequest.uid, collection.uid, message.content);
        });
      }
    }

    // 第 328-360 行：获取 SSL 配置并启动连接
    // wsClient 来自 @usebruno/requests 包，源文件见下文
    const certsAndProxyConfig = await getCertsAndProxyConfig(...);
    const sslOptions = { rejectUnauthorized: preferencesUtil.shouldVerifyTls(), ... };

    await wsClient.startConnection({
      request: preparedRequest, collection,
      options: { timeout: settings.timeout, keepAlive: settings.keepAliveInterval > 0, keepAliveInterval: settings.keepAliveInterval, sslOptions }
    });

    // 第 362 行：发送请求事件到时间线
    sendEvent('main:ws:request', preparedRequest.uid, collection.uid, requestSent);
    return { success: true };
  }
);
```

**WebSocket 关键实现位置**（关键修正）：
- `WsClient` 导出 → `packages/bruno-requests/src/index.ts:3`
- `WsClient` 源实现 → `packages/bruno-requests/src/ws/ws-client.js`
- `prepareWsRequest`（WebSocket 请求预处理）→ `packages/bruno-electron/src/ipc/network/ws-event-handlers.js:27-215`（在同一文件内定义）

---

## 三、响应展示与错误归一

### 3.1 响应状态存储与归一化

#### 3.1.1 统一响应接收 Reducer

```javascript
// 文件: packages/bruno-app/src/providers/ReduxStore/slices/collections/index.js:527-650
responseReceived: (state, action) => {
  // 找到对应的 collection 和 item，更新 response 状态
  // 支持：HTTP/GraphQL 同步响应、gRPC/WebSocket 事件驱动响应
  const { itemUid, collectionUid, response, requestSent, responseType } = action.payload;
  const collection = state.collections.find((c) => c.uid === collectionUid);
  if (collection) {
    const item = collection.items.find((i) => i.uid === itemUid);
    if (item) {
      item.response = response;       // 归一化响应数据
      item.requestSent = requestSent; // 原始请求记录
      item.requestState = 'received'; // 统一状态标识
    }
  }
};
```

### 3.2 响应展示面板架构

#### 3.2.1 ResponsePane：基础响应面板

```javascript
// 文件: packages/bruno-app/src/components/ResponsePane/index.js
const ResponsePane = ({ item, collection }) => {
  // 第 153-200 行：Tab 面板渲染
  const getTabPanel = (tab) => {
    switch (tab) {
      case 'response':
        // 流式响应使用 WSMessagesList
        const isStream = item.response?.stream ?? false;
        if (isStream) {
          return <WSMessagesList order={-1} messages={item.response.data} />;
        }
        return <QueryResult ... />;  // 普通响应展示
      case 'headers':
        return <ResponseHeaders headers={response.headers} />;
      case 'timeline':
        return <Timeline collection={collection} item={item} activeTabUid={activeTabUid} />;
      case 'tests':
        return <TestResults item={item} ... />;
    }
  };

  // 第 262-267 行：统一状态展示组件
  <StatusCode status={response.status} isStreaming={item.response?.stream?.running} />
  {item.response?.stream?.running
    ? <ResponseStopWatch startMillis={response.duration} />
    : <ResponseTime duration={response.duration} />}
  <ResponseSize size={responseSize} />
};
```

**通用组件位置**：
- `StatusCode` → `packages/bruno-app/src/components/ResponsePane/StatusCode/index.js`
- `ResponseTime` → `packages/bruno-app/src/components/ResponsePane/ResponseTime/index.js`
- `ResponseStopWatch` → `packages/bruno-app/src/components/ResponsePane/ResponseStopWatch/index.js`
- `ResponseSize` → `packages/bruno-app/src/components/ResponsePane/ResponseSize/index.js`
- `Timeline` → `packages/bruno-app/src/components/ResponsePane/Timeline/index.js`
- `QueryResult` → `packages/bruno-app/src/components/ResponsePane/QueryResult/index.js`
- `ResponseHeaders` → `packages/bruno-app/src/components/ResponsePane/ResponseHeaders/index.js`
- `TestResults` → `packages/bruno-app/src/components/ResponsePane/TestResults/index.js`

#### 3.2.2 GrpcResponsePane：gRPC 专用面板

```javascript
// 文件: packages/bruno-app/src/components/ResponsePane/GrpcResponsePane/index.js
const GrpcResponsePane = ({ item, collection, response }) => {
  // gRPC 特有组件
  <GrpcStatusCode status={response.status} />              // gRPC 状态码映射
  <GrpcResponseHeaders headers={response.metadata} />      // Metadata 展示
  <GrpcQueryResult messages={response.messages} />         // 流式消息列表
  <ResponseTrailers trailers={response.trailers} />        // Trailers 展示
  <GrpcError error={response.error} />                     // gRPC 错误展示
};
```

**gRPC 专用组件位置**：
- `GrpcStatusCode` → `packages/bruno-app/src/components/ResponsePane/GrpcResponsePane/GrpcStatusCode/index.js`
- `GrpcResponseHeaders` → `packages/bruno-app/src/components/ResponsePane/GrpcResponsePane/GrpcResponseHeaders/index.js`
- `ResponseTrailers` → `packages/bruno-app/src/components/ResponsePane/GrpcResponsePane/ResponseTrailers/index.js`
- `GrpcQueryResult` → `packages/bruno-app/src/components/ResponsePane/GrpcResponsePane/GrpcQueryResult/index.js`
- `GrpcError` → `packages/bruno-app/src/components/ResponsePane/GrpcResponsePane/GrpcError/index.js`

#### 3.2.3 WSResponsePane：WebSocket 专用面板

```javascript
// 文件: packages/bruno-app/src/components/ResponsePane/WsResponsePane/index.js
const WSResponsePane = ({ item, collection, response }) => {
  // WebSocket 特有组件
  <WSStatusCode statusCode={response.statusCode} />        // WebSocket 关闭码映射
  <WSMessagesList messages={response.messages} />           // 收发消息列表
  <WSResponseHeaders headers={response.headers} />          // 握手响应头
};
```

**WebSocket 专用组件位置**：
- `WSStatusCode` → `packages/bruno-app/src/components/ResponsePane/WsResponsePane/WSStatusCode/index.js`
- `WSMessagesList` → `packages/bruno-app/src/components/ResponsePane/WsResponsePane/WSMessagesList/index.js`
- `WSResponseHeaders` → `packages/bruno-app/src/components/ResponsePane/WsResponsePane/WSResponseHeaders/index.js`

### 3.3 错误归一化处理

#### 3.3.1 HTTP/GraphQL 错误归一化

```javascript
// 文件: packages/bruno-app/src/providers/ReduxStore/slices/collections/actions.js:613-645
.catch((err) => {
  const request = itemCopy.draft?.request || itemCopy.request;
  const requestSent = request ? { url: request.url, method: request.method } : undefined;

  // 取消请求的特殊处理
  if (err && err.message === 'Error invoking remote method \'send-http-request\': Error: Request cancelled') {
    dispatch(responseReceived({ itemUid, collectionUid, response: null, requestSent }));
    return;
  }

  // 统一错误响应结构
  const errorResponse = {
    status: 'Error',
    isError: true,
    error: err.message ?? 'Something went wrong',
    size: 0,
    duration: 0
  };
  dispatch(responseReceived({ itemUid, collectionUid, response: errorResponse, requestSent }));
});
```

#### 3.3.2 错误通知组件

```javascript
// 文件: packages/bruno-app/src/components/ResponsePane/NetworkError/index.js
const NetworkError = ({ onClose }) => {
  // Toast 通知形式展示网络错误
  return (
    <div className="max-w-md w-full bg-white shadow-lg rounded-lg pointer-events-auto flex bg-red-100">
      <div className="flex-1 w-0 p-4">
        <div className="flex items-start">
          <div className="ml-3 flex-1">
            <p className="font-medium text-red-800">Network Error</p>
          </div>
        </div>
      </div>
      <div className="flex">
        <button onClick={onClose}>Close</button>
      </div>
    </div>
  );
};
```

**调用位置**：`RequestTabPanel/index.js:447` - 在 sendRequest catch 中通过 toast.custom 展示

### 3.4 事件驱动的流式响应更新

gRPC 和 WebSocket 采用事件驱动架构更新响应状态：

```javascript
// 事件监听（在应用初始化时注册）
// WebSocket 事件: main:ws:connected, main:ws:message, main:ws:closed, main:ws:error
// gRPC 事件: grpc:response, grpc:stream-message, grpc:closed, grpc:error

// 事件响应时调用相同的 responseReceived reducer
// 实现流式数据的实时更新
```

---

## 四、架构总结

### 4.1 实际调用链路图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                               UI 层 (Renderer)                               │
├─────────────────────────────────────────────────────────────────────────────┤
│  RequestTabPanel (统一入口)                                                  │
│    ├── handleRun (第 428 行) ─────────────────────────────────────────────┐  │
│    ├── QueryUrl (HTTP/GraphQL)                                            │  │
│    ├── GrpcQueryUrl                                                        │  │
│    └── WsQueryUrl                                                          │  │
│                                                                             │  │
└─────────────────────────────────────────────────────────────────────────────┘  │
                                                                                   │
┌─────────────────────────────────────────────────────────────────────────────┐  │
│                            Redux Action 层                                   │  │
├─────────────────────────────────────────────────────────────────────────────┤  │
│  sendRequest (actions.js:528)                                               │  │
│    ├── HTTP/GraphQL → sendNetworkRequest → sendHttpRequest ───────────────┼──┘
│    ├── gRPC         → sendGrpcRequest → startGrpcRequest ──────────────────┤
│    └── WebSocket    → sendWsRequest → connectWS/queueWsMessage ────────────┤
└─────────────────────────────────────────────────────────────────────────────┘
                                                                                   │
┌─────────────────────────────────────────────────────────────────────────────┐  │
│                              IPC 通信层                                      │  │
├─────────────────────────────────────────────────────────────────────────────┤  │
│  ipcRenderer.invoke(...)                                                     │  │
│    ├── 'send-http-request'                                                   │  │
│    ├── 'grpc:start-connection'                                               │  │
│    └── 'renderer:ws:start-connection'                                       │  │
└─────────────────────────────────────────────────────────────────────────────┘  │
                                                                                   │
┌─────────────────────────────────────────────────────────────────────────────┐  │
│                          主进程层 (Main - Electron)                          │  │
├─────────────────────────────────────────────────────────────────────────────┤  │
│  ipcMain.handle(...)                                                         │  │
│    ├── send-http-request (network/index.js:1205)                           │  │
│    │   └── runRequest ──► prepareRequest ──► configureRequest ──► axios    │  │
│    ├── grpc:start-connection (grpc-event-handlers.js:155)                 │  │
│    │   └── prepareGrpcRequest ──► configureRequest ──► GrpcClient          │  │
│    └── renderer:ws:start-connection (ws-event-handlers.js:303)            │  │
│        └── prepareWsRequest ──► WsClient.startConnection()                 │  │
└─────────────────────────────────────────────────────────────────────────────┘
                                                                                   │
┌─────────────────────────────────────────────────────────────────────────────┐  │
│                         @usebruno/requests 核心库层                           │  │
├─────────────────────────────────────────────────────────────────────────────┤  │
│  packages/bruno-requests/src/                                                │  │
│    ├── grpc/grpc-client.js ←───┐                                            │  │
│    ├── ws/ws-client.js ←───────┤  协议客户端实现                              │  │
│    ├── network/axios-instance.ts ←┘                                          │  │
│    └── index.ts (统一导出入口)                                              ◄──┘
└─────────────────────────────────────────────────────────────────────────────┘
                                                                                   │
┌─────────────────────────────────────────────────────────────────────────────┐  │
│                            响应归一化层                                       │  │
├─────────────────────────────────────────────────────────────────────────────┤  │
│  HTTP/GraphQL: Promise.then() ──► responseReceived reducer                ◄──┘
│  gRPC/WebSocket: 事件监听 ──► responseReceived reducer                      │
│  错误处理: catch() ──► errorResponse ──► responseReceived reducer          │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 关键设计原则

| 原则 | 实现方式 | 文件位置 |
|------|---------|---------|
| **入口统一** | 所有协议通过相同的 handleRun → sendRequest 路径 | RequestTabPanel.js:428 |
| **职责分离** | 协议分发在 Action 层，传输实现封装在 `@usebruno/requests` | network/index.js 各分支 |
| **状态归一** | 所有协议响应最终汇入同一个 responseReceived reducer | collections/index.js:527 |
| **展示复用** | StatusCode、Timeline 等组件跨协议共享，专用面板继承扩展 | ResponsePane/ 目录 |
| **错误一致** | 所有协议错误都映射为相同结构，使用统一 NetworkError toast | actions.js:613-645 |

### 4.3 configureRequest 实现对比

| 协议 | 文件位置 | 主要职责 |
|------|---------|---------|
| HTTP/GraphQL | `bruno-electron/src/ipc/network/index.js:101` | 代理配置、SSL/TLS、OAuth1/OAuth2、NTLM、Axios 实例配置 |
| gRPC | `bruno-electron/src/ipc/network/prepare-grpc-request.js:32` | OAuth2 token 获取、token 放置位置（header/query） |

### 4.4 核心客户端库位置汇总（@usebruno/requests）

| 组件 | 文件路径 | 功能 |
|------|---------|------|
| **GrpcClient** | `packages/bruno-requests/src/grpc/grpc-client.js` | gRPC 连接管理、流式调用、消息收发 |
| **WsClient** | `packages/bruno-requests/src/ws/ws-client.js` | WebSocket 连接管理、消息队列、心跳检测 |
| **axios-instance** | `packages/bruno-requests/src/network/axios-instance.ts` | HTTP 客户端代理配置、Keep-Alive、证书 |
| **统一导出** | `packages/bruno-requests/src/index.ts` | 所有客户端类和工具函数的导出入口 |

---

*报告生成时间：2026-05-12*
*基于 Bruno v118 实际代码实现分析*
