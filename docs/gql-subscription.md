# GraphQL Subscription 长连接机制解析

本文档详细解析 Bruno 中 GraphQL Subscription（基于 WebSocket 协议）的长连接实现机制，涵盖握手协商、消息分发、断线重连三个核心模块。

## 目录
1. [整体架构](#整体架构)
2. [握手与协议协商](#握手与协议协商)
3. [消息分发机制](#消息分发机制)
4. [断线重连策略](#断线重连策略)

---

## 整体架构

Bruno 的 WebSocket/GraphQL Subscription 系统采用三层架构设计：

```
┌─────────────────────────────────────────────────────────────┐
│                     Renderer Layer (UI)                      │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  WsQueryUrl  │  WSRequestPane  │  Response Panel       │  │
│  └───────────────────────────────────────────────────────┘  │
│                              │                               │
│                              ▼ IPC Events                   │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              useWsEventListeners (React Hook)          │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼ IPC Communication
┌─────────────────────────────────────────────────────────────┐
│                     Main Process (Node.js)                    │
│  ┌───────────────────────────────────────────────────────┐  │
│  │           registerWsEventHandlers (ipcMain)           │  │
│  └───────────────────────────────────────────────────────┘  │
│                              │                               │
│                              ▼                               │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                   WsClient (Core)                      │  │
│  │  - Connection Management  │  Message Queue           │  │
│  │  - Keep-Alive Ping       │  Event Emitter           │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼ WebSocket Protocol
┌─────────────────────────────────────────────────────────────┐
│                     GraphQL Server                           │
│         (graphql-ws / subscriptions-transport-ws)           │
└─────────────────────────────────────────────────────────────┘
```

**关键文件位置：**
- `packages/bruno-requests/src/ws/ws-client.js` - 核心 WebSocket 客户端
- `packages/bruno-electron/src/ipc/network/ws-event-handlers.js` - IPC 事件处理器
- `packages/bruno-app/src/utils/network/ws-event-listeners.js` - React 事件监听 Hook
- `packages/bruno-app/src/components/RequestPane/WsQueryUrl/index.js` - 连接控制 UI

---

## 握手与协议协商

### 1. 连接初始化流程

**Step 1: 触发连接**
```javascript
// 文件: packages/bruno-app/src/components/RequestPane/WsQueryUrl/index.js
const handleConnect = async () => {
  dispatch(wsConnectOnly(item, collection.uid));
  previousDeboundedInterpolatedURL.current = debouncedInterpolatedURL;
};
```

**Step 2: 准备请求参数**
```javascript
// 文件: packages/bruno-electron/src/ipc/network/ws-event-handlers.js:27-283
const prepareWsRequest = async (item, collection, environment, runtimeVariables) => {
  // 1. 合并层级 Headers (Folder -> Request)
  // 2. 处理认证: OAuth2, API Key, Basic Auth 等
  // 3. 变量插值: {{variable}} -> 实际值
  // 4. 处理 Sec-WebSocket-Protocol 头
  // 5. SSL/TLS 证书配置
};
```

**Step 3: 建立 WebSocket 连接**
```javascript
// 文件: packages/bruno-requests/src/ws/ws-client.js:98-161
async startConnection({ request, collection, options = {} }) {
  const { url, headers } = request;
  
  // 解析 WebSocket 协议
  const protocols = []
    .concat([headers['Sec-WebSocket-Protocol'], headers['sec-websocket-protocol']])
    .filter(Boolean)
    .map((d) => d.split(','))
    .flat()
    .map((d) => d.trim());

  // 创建 WebSocket 实例
  const wsConnection = new ws.WebSocket(parsedUrl.fullUrl, protocols, {
    headers,
    handshakeTimeout: validTimeout,
    followRedirects: true,
    rejectUnauthorized: sslOptions.rejectUnauthorized,
    ca: sslOptions.ca,
    cert: sslOptions.cert,
    key: sslOptions.key,
    pfx: sslOptions.pfx,
    passphrase: sslOptions.passphrase
  });

  // 设置事件处理器
  this.#setupWsEventHandlers(wsConnection, requestId, collectionUid, options);
  this.#addConnection(requestId, collectionUid, wsConnection);
  
  // 触发 connecting 事件
  this.eventCallback('main:ws:connecting', requestId, collectionUid);
}
```

### 2. 协议协商机制

**支持的 WebSocket 子协议：**
- `graphql-transport-ws` (graphql-ws 协议)
- `graphql-ws` (旧版 subscriptions-transport-ws 协议)
- 自定义协议通过 `Sec-WebSocket-Protocol` Header 指定

**协议协商流程：**
1. 用户在 Request Headers 中配置 `Sec-WebSocket-Protocol: graphql-transport-ws`
2. `prepareWsRequest` 提取并规范化协议头（第 61-70 行）
3. `WsClient.startConnection` 将协议数组传递给 WebSocket 构造函数
4. 服务端返回 `101 Switching Protocols` 响应，包含协商后的协议
5. 通过 `upgrade` 事件通知前端握手完成

```javascript
// 文件: packages/bruno-requests/src/ws/ws-client.js:335-342
ws.on('upgrade', (response) => {
  this.eventCallback('main:ws:upgrade', requestId, collectionUid, {
    type: 'info',
    timestamp: Date.now(),
    seq: seq.next(requestId, collectionUid),
    headers: { ...response.headers }
  });
});
```

### 3. GraphQL 订阅协议消息

虽然 Bruno 没有内置 graphql-ws 协议的自动化处理，但支持用户手动发送协议消息：

**典型 GraphQL Subscription 协议交互：**
```
Client → Server: {"type":"connection_init","payload":{"token":"..."}}
Server → Client: {"type":"connection_ack"}
Client → Server: {"id":"1","type":"subscribe","payload":{"query":"subscription { ... }"}}
Server → Client: {"id":"1","type":"next","payload":{"data":{...}}}
Server → Client: {"id":"1","type":"complete"}
```

---

## 消息分发机制

### 1. 消息队列设计

**队列目的：** 解决连接建立前的消息发送时序问题，确保消息在连接 open 后按顺序发送。

```javascript
// 文件: packages/bruno-requests/src/ws/ws-client.js:167-190
queueMessage(requestId, collectionUid, message, format = 'raw') {
  const connectionMeta = this.activeConnections.get(requestId);

  const mqKey = this.#getMessageQueueId(requestId);
  this.messageQueues[mqKey] ||= [];
  this.messageQueues[mqKey].push({ message, format });

  // 如果连接已打开，立即刷新队列
  if (connectionMeta && connectionMeta.connection && 
      connectionMeta.connection.readyState === WebSocket.OPEN) {
    this.#flushQueue(requestId, collectionUid);
    return;
  }
}

#flushQueue(requestId, collectionUid) {
  const mqKey = this.#getMessageQueueId(requestId);
  if (!(mqKey in this.messageQueues)) return;
  
  while (this.messageQueues[mqKey].length > 0) {
    const { message, format } = this.messageQueues[mqKey].shift();
    this.sendMessage(requestId, collectionUid, message, format);
  }
}
```

**队列刷新触发点：**
- WebSocket `open` 事件触发时（第 304-306 行）
- 调用 `queueMessage` 时若连接已建立

### 2. 消息发送流程

```javascript
// 文件: packages/bruno-requests/src/ws/ws-client.js:198-225
sendMessage(requestId, collectionUid, message, format = 'raw') {
  const connectionMeta = this.activeConnections.get(requestId);

  if (connectionMeta.connection && 
      connectionMeta.connection.readyState === WebSocket.OPEN) {
    const payload = normalizeMessageByFormat(message, format);

    connectionMeta.connection.send(payload, (error) => {
      if (error) {
        this.eventCallback('main:ws:error', requestId, collectionUid, { error });
      } else {
        // 消息发送成功，通知 UI
        this.eventCallback('main:ws:message', requestId, collectionUid, {
          message: payload,
          messageHexdump: hexdump(payload),
          type: 'outgoing',
          seq: seq.next(requestId, collectionUid),
          timestamp: Date.now()
        });
      }
    });
  } else {
    const error = new Error('WebSocket connection not available or not open');
    this.eventCallback('main:ws:error', requestId, collectionUid, {
      error: error.message
    });
  }
}
```

### 3. 消息接收与分发

**接收事件处理器：**
```javascript
// 文件: packages/bruno-requests/src/ws/ws-client.js:344-364
ws.on('message', (data) => {
  try {
    // 尝试解析为 JSON（GraphQL 消息通常为 JSON）
    const message = JSON.parse(data.toString());
    this.eventCallback('main:ws:message', requestId, collectionUid, {
      message,
      messageHexdump: hexdump(Buffer.from(data)),
      type: 'incoming',
      seq: seq.next(requestId, collectionUid),
      timestamp: Date.now()
    });
  } catch (error) {
    // 解析失败，作为原始字符串发送
    this.eventCallback('main:ws:message', requestId, collectionUid, {
      message: data.toString(),
      messageHexdump: hexdump(data),
      type: 'incoming',
      seq: seq.next(requestId, collectionUid),
      timestamp: Date.now()
    });
  }
});
```

**IPC 事件分发到 UI：**
```javascript
// 文件: packages/bruno-app/src/utils/network/ws-event-listeners.js:45-53
const removeWsMessageListener = ipcRenderer.on('main:ws:message', 
  (requestId, collectionUid, eventData) => {
    dispatch(wsResponseReceived({
      itemUid: requestId,
      collectionUid: collectionUid,
      eventType: 'message',
      eventData: eventData
    }));
  }
);
```

**Redux Store 更新：**
- 事件通过 `wsResponseReceived` action 分发
- Response Panel 订阅 store 变化，实时追加消息
- 消息序列号 `seq` 确保 UI 显示顺序正确

---

## 断线重连策略

### 1. URL 变更自动重连

**检测机制：**
```javascript
// 文件: packages/bruno-app/src/components/RequestPane/WsQueryUrl/index.js:117-122
useEffect(() => {
  if (connectionStatus !== 'connected') return;
  if (previousDeboundedInterpolatedURL.current === debouncedInterpolatedURL) return;
  if (debouncedInterpolatedURL === '') return;
  handleReconnect();
}, [debouncedInterpolatedURL, connectionStatus]);
```

**重连实现：**
```javascript
// 文件: packages/bruno-app/src/components/RequestPane/WsQueryUrl/index.js:81-91
const handleReconnect = async (e) => {
  e && e.stopPropagation();
  try {
    handleDisconnect(e, false);  // 先关闭旧连接
    setTimeout(() => {
      handleConnect(e, false);   // 2秒后建立新连接
    }, 2000);
  } catch (err) {
    console.error('Failed to re-connect WebSocket connection', err);
  }
};
```

### 2. Keep-Alive 心跳机制

**心跳配置：**
- 用户可在 Settings 面板配置 `Keep Alive Interval`（毫秒）
- 值为 0 表示禁用心跳
- 心跳通过 WebSocket Ping 帧实现

**心跳实现：**
```javascript
// 文件: packages/bruno-requests/src/ws/ws-client.js:307-313
if (options.keepAlive) {
  const handle = setInterval(() => {
    ws.isAlive = false;
    ws.ping();  // 发送 Ping 帧
  }, options.keepAliveInterval);

  this.connectionKeepAlive.set(requestId, handle);
}
```

**Pong 响应处理（ws 库内置）：**
- 服务端返回 Pong 帧时，`ws` 库自动更新连接状态
- 若超时未收到 Pong，触发 `error` 或 `close` 事件

### 3. 连接状态管理

**状态定义：**
```javascript
// 文件: packages/bruno-app/src/components/RequestPane/WsQueryUrl/index.js:20-24
const CONNECTION_STATUS = {
  CONNECTING: 'connecting',
  CONNECTED: 'connected',
  DISCONNECTED: 'disconnected'
};
```

**状态轮询：**
```javascript
// 文件: packages/bruno-app/src/components/RequestPane/WsQueryUrl/index.js:26-38
const useWsConnectionStatus = (requestId) => {
  const [connectionStatus, setConnectionStatus] = useState(CONNECTION_STATUS.DISCONNECTED);
  useEffect(() => {
    const checkConnectionStatus = async () => {
      const result = await getWsConnectionStatus(requestId);
      setConnectionStatus(result?.status ?? CONNECTION_STATUS.DISCONNECTED);
    };
    checkConnectionStatus();
    const interval = setInterval(checkConnectionStatus, 2000);  // 每2秒轮询
    return () => clearInterval(interval);
  }, [requestId]);
  return [connectionStatus, setConnectionStatus];
};
```

### 4. 主动清理机制

**连接关闭清理：**
```javascript
// 文件: packages/bruno-requests/src/ws/ws-client.js:409-430
#removeConnection(requestId) {
  // 1. 清除 Keep-Alive 定时器
  if (this.connectionKeepAlive.has(requestId)) {
    clearInterval(this.connectionKeepAlive.get(requestId));
    this.connectionKeepAlive.delete(requestId);
  }

  // 2. 清空消息队列
  const mqId = this.#getMessageQueueId(requestId);
  if (mqId in this.messageQueues) {
    this.messageQueues[mqId] = [];
  }

  // 3. 从活跃连接 Map 中移除
  if (this.activeConnections.has(requestId)) {
    this.activeConnections.delete(requestId);
    
    // 4. 通知 UI 连接列表变更
    this.eventCallback('main:ws:connections-changed', {
      type: 'removed',
      requestId,
      activeConnectionIds: this.getActiveConnectionIds()
    });
  }
}
```

### 5. 重连时的消息保留策略

**当前行为：**
- 断线后消息队列被清空（`#removeConnection` 第 415-418 行）
- 重连后需要用户重新发送订阅消息

**建议的增强方案（可扩展）：**
1. 断线时保存未发送的消息队列
2. 重连成功后自动重发订阅消息
3. 对 GraphQL subscription 类型自动发送 `connection_init` 和恢复订阅

---

## 总结

### 核心优势
1. **分层架构清晰**：Renderer ↔ Main Process ↔ Network 三层分离
2. **消息队列可靠**：解决连接建立前后的时序问题
3. **灵活的协议支持**：不绑定特定 GraphQL 协议，用户可自由配置
4. **实时事件驱动**：基于 IPC 的事件分发，UI 响应及时

### 可改进点
1. **自动重连增强**：目前仅支持 URL 变更触发，可增加网络恢复检测
2. **GraphQL 协议自动化**：内置 graphql-ws 协议处理，自动处理 connection_init、心跳、重连后恢复订阅
3. **消息持久化**：断线时保存订阅状态，重连后自动恢复

---

**文档版本：** 1.0  
**最后更新：** 2025-05-12  
**适用版本：** Bruno v2.x+
