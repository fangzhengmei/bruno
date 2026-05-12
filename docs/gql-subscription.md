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

**关键文件位置**：
- `packages/bruno-requests/src/ws/ws-client.js` - 核心 WebSocket 客户端
- `packages/bruno-electron/src/ipc/network/ws-event-handlers.js` - IPC 事件处理器
- `packages/bruno-app/src/utils/network/ws-event-listeners.js` - React 事件监听 Hook
- `packages/bruno-app/src/components/RequestPane/WsQueryUrl/index.js` - 连接控制 UI
- `packages/bruno-app/src/providers/ReduxStore/slices/collections/index.js` - Redux 状态管理

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

**支持的 WebSocket 子协议**：
- `graphql-transport-ws` (graphql-ws 协议)
- `graphql-ws` (旧版 subscriptions-transport-ws 协议)
- 自定义协议通过 `Sec-WebSocket-Protocol` Header 指定

**协议协商流程**：
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

**典型 GraphQL Subscription 协议交互**：
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

**队列目的**：解决连接建立前的消息发送时序问题，确保消息在连接 open 后按顺序发送。

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

**队列刷新触发点**：
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

### 3. 消息接收与响应面板追加完整路径

**完整事件链路**（从服务端到 UI 渲染）：

```
Step 1: WebSocket 接收数据
        ↓ (ws.on('message'))
Step 2: WsClient 触发 eventCallback('main:ws:message', ...)
        ↓ (sendEvent)
Step 3: Main 进程通过 webContents.send 发送到 Renderer
        ↓ (ipcRenderer.on)
Step 4: useWsEventListeners Hook 监听到事件
        ↓ (dispatch)
Step 5: wsResponseReceived reducer 处理
        ↓ (concat to responses)
Step 6: item.response.responses 数组追加消息
        ↓ (selector / re-render)
Step 7: ResponsePane 组件订阅状态变化，实时渲染
```

**Step 1-2: 接收与事件触发**
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

**Step 3: IPC 跨进程传递**
```javascript
// 文件: packages/bruno-electron/src/ipc/network/ws-event-handlers.js:291-298
const sendEvent = (eventName, ...args) => {
  if (window && !window.isDestroyed() && window.webContents && !window.webContents.isDestroyed()) {
    window.webContents.send(eventName, ...args);
  }
};
```

**Step 4: Renderer 监听与 Dispatch**
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

**Step 5-6: Redux Reducer 追加消息**
```javascript
// 文件: packages/bruno-app/src/providers/ReduxStore/slices/collections/index.js:3437-3462
wsResponseReceived: (state, action) => {
  const { itemUid, collectionUid, eventType, eventData } = action.payload;
  const collection = findCollectionByUid(state.collections, collectionUid);
  const item = findItemInCollection(collection, itemUid);
  
  const currentResponse = item.response || initiatedWsResponse;
  
  switch (eventType) {
    case 'message':
      // 关键: 使用 concat 追加新消息到 responses 数组，保留历史消息
      updatedResponse.responses = (currentResponse?.responses || []).concat(eventData);
      break;
    // ... 其他事件处理
  }
  
  item.response = updatedResponse;
}
```

**Step 7: UI 渲染**
```javascript
// 文件: packages/bruno-app/src/components/ResponsePane/WsResponsePane/WSMessagesList/index.js:177
const WSMessagesList = ({ messages = [] }) => {
  // ...
  // 直接按数组顺序渲染，不进行任何排序
  return (
    <Virtuoso
      data={messages}  // 直接使用传入的数组顺序
      itemContent={renderItem}
      computeItemKey={(_, msg) => msg.seq ?? msg.timestamp}  // 仅用作 React key
      // ...
    />
  );
};
```

**关键要点**：
- ResponsePane 通过 `useSelector` 订阅 `item.response.responses`
- 数组更新触发组件重渲染
- **直接按数组追加顺序渲染**，不进行任何排序操作
- `seq` 仅用作 React 的 `key` 属性（避免列表元素复用优化）

### 4. seq 字段的真实作用与局限

**seq 的生成机制**：
```javascript
// 文件: packages/bruno-requests/src/ws/ws-client.js:52-78
const createSequencer = () => {
  const seq = {};

  const nextSeq = (requestId, collectionId) => {
    seq[requestId] ||= {};
    seq[requestId][collectionId] ||= 0;
    return ++seq[requestId][collectionId];  // 按 requestId + collectionId 维度自增
  };

  const clean = (requestId, collectionId = undefined) => {
    // 清理计数器
  };

  return { next: nextSeq, clean };
};
```

**seq 的真实作用**：
1. **React 列表渲染优化**：在 WSMessagesList 中用作 `computeItemKey`，帮助 React 识别稳定的列表元素标识
2. **事件标识**：每个 WebSocket 事件（message/open/close/error/upgrade）都带有唯一 seq
3. **调试追踪**：便于在日志或问题时追踪消息流向

**seq 的局限**：
1. **不用于排序**：渲染时直接使用数组顺序，不基于 seq 做任何排序
2. **作用域有限**：每个 requestId + collectionId 组合维度自增，跨请求/跨集合不连续
3. **断线重置**：连接关闭时调用 `seq.clean()`，重连后 seq 从 1 重新计数
4. **非严格单调**：仅保证同一连接内事件发生时 seq 递增，但不保证绝对连续（因为 clean 可能被调用）

---

## 断线重连策略

### 1. 网络断开后的行为

**重要澄清**：Bruno 当前实现中**网络断开后不会自动重连**。

**断开检测机制**：
- 网络断开时，WebSocket 触发 `close` 事件
- 触发 `main:ws:close` 事件，UI 更新状态为 `CLOSED`
- 从 `activeConnections` Map 中移除连接
- **清空该连接的消息队列**（#removeConnection 第 415-418 行）

```javascript
// 文件: packages/bruno-requests/src/ws/ws-client.js:366-375
ws.on('close', (code, reason) => {
  this.eventCallback('main:ws:close', requestId, collectionUid, {
    code,
    reason: Buffer.from(reason).toString(),
    seq: seq.next(requestId, collectionUid),
    timestamp: Date.now()
  });
  seq.clean(requestId, collectionUid);
  this.#removeConnection(requestId);  // 这里会清空消息队列
});
```

### 2. 仅 URL 变化触发的重连

**唯一自动重连触发点**：URL（包括变量插值后的 URL）发生变化。

```javascript
// 文件: packages/bruno-app/src/components/RequestPane/WsQueryUrl/index.js:117-122
useEffect(() => {
  if (connectionStatus !== 'connected') return;  // 未连接时不触发
  if (previousDeboundedInterpolatedURL.current === debouncedInterpolatedURL) return;
  if (debouncedInterpolatedURL === '') return;
  handleReconnect();  // 仅当 URL 变化时才重连
}, [debouncedInterpolatedURL, connectionStatus]);
```

**重连实现**：
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

### 3. 重连后需要手动重发 Subscription 消息

**关键限制**：
1. 重连建立的是**全新的 WebSocket 连接**，与旧连接无状态关联
2. 断线时消息队列已被清空，重连后不会自动重发
3. GraphQL 协议层的 `connection_init`、`subscribe` 等消息需要用户手动重新发送
4. 服务端不会记住之前的订阅状态

**用户操作流程**：
```
网络断开 → URL 变化触发重连 → 新连接建立成功
       ↓
需要用户手动点击发送:
  1. {"type":"connection_init"} 重新初始化连接
  2. {"id":"1","type":"subscribe",...} 重新建立订阅
```

### 4. Keep-Alive 心跳机制的真实行为

**纠正之前的错误表述**：

Bruno 的 Keep-Alive 机制**仅发送 Ping 帧，不会主动检测超时或关闭连接**。

```javascript
// 文件: packages/bruno-requests/src/ws/ws-client.js:307-313
if (options.keepAlive) {
  const handle = setInterval(() => {
    ws.isAlive = false;  // 设置标记但从未检查
    ws.ping();           // 仅发送 Ping 帧
  }, options.keepAliveInterval);

  this.connectionKeepAlive.set(requestId, handle);
}
```

**真实行为解析**：
1. **发送 Ping**：按配置间隔定期发送 Ping 帧到服务端
2. **接收 Pong**：ws 库内部自动响应 Pong，但 Bruno 代码中**没有监听 Pong 事件**
3. **isAlive 标记**：设置了 `ws.isAlive = false`，但**从未在超时后检查该值**
4. **无主动关闭**：没有实现 "若 N 秒未收到 Pong 则关闭连接" 的逻辑

**心跳的实际作用**：
- 防止网络中间设备（如防火墙、NAT 网关）因连接空闲而断开
- 被动检测连接状态（依赖底层 TCP 超时或 WebSocket close 事件）
- **不具备主动断线检测和自动重连能力**

### 5. 连接状态管理

**状态定义**：
```javascript
// 文件: packages/bruno-app/src/components/RequestPane/WsQueryUrl/index.js:20-24
const CONNECTION_STATUS = {
  CONNECTING: 'connecting',
  CONNECTED: 'connected',
  DISCONNECTED: 'disconnected'
};
```

**状态轮询**：
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

### 6. 主动清理机制

**连接关闭清理**：
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
    this.messageQueues[mqId] = [];  // 队列被清空，重连后不会自动重发
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

---

## 总结

### 核心能力现状
| 能力 | 支持状态 | 说明 |
|------|----------|------|
| WebSocket 基础连接 | ✅ | 支持自定义 Header、协议、SSL |
| 消息队列缓冲 | ✅ | 连接建立前缓存消息 |
| 消息实时追加 | ✅ | 响应面板持续追加新消息 |
| URL 变更自动重连 | ✅ | 变量插值变化触发重连 |
| Keep-Alive 心跳 | ⚠️ | 仅发送 Ping，无超时检测 |
| 网络断开自动重连 | ❌ | 需要 URL 变化或手动触发 |
| 重连后自动恢复订阅 | ❌ | 需要手动重新发送协议消息 |

### 可改进点
1. **自动重连增强**：
   - 监听网络状态变化（online/offline 事件）
   - 实现指数退避重连策略
   - 断线时保留未发送消息队列

2. **GraphQL 协议自动化**：
   - 内置 graphql-ws 协议处理器
   - 自动发送 `connection_init` 保持连接
   - 重连后自动恢复之前的订阅
   - 管理订阅 ID 映射

3. **心跳机制完善**：
   - 监听 Pong 事件更新 `isAlive` 状态
   - 实现超时检测（如 3 次心跳未响应则关闭）
   - 触发自动重连逻辑

---

**文档版本**：1.1  
**最后更新**：2025-05-12  
**适用版本**：Bruno v2.x+
