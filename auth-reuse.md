# Bruno 鉴权复用机制详解

本报告详细分析 Bruno API 客户端的三大核心鉴权复用机制：Cookie 罐持久化、OAuth2 多 Grant Type 适配、跨请求鉴权复用。

---

## 一、Cookie 罐持久化机制

### 1.1 架构概览

Cookie 系统采用 **双层架构**：内存 Cookie Jar + 磁盘持久化存储，基于 `tough-cookie` 库和 `electron-store` 实现。

```
内存层 (tough-cookie)          磁盘层 (electron-store)
┌────────────────────┐        ┌────────────────────┐
│  CookieJar         │◀──────▶│  加密 Cookie 存储  │
│  - 按 Domain 组织  │        │  - 按 Domain 分组  │
│  - 过期自动清理    │        │  - AES 加密存储    │
└────────────────────┘        └────────────────────┘
          ▲
          │ 5s 防抖写入
          ▼
┌────────────────────┐
│  请求/响应 拦截器  │
└────────────────────┘
```

### 1.2 核心实现模块

| 模块 | 文件路径 | 职责 |
|------|---------|------|
| Cookie 核心操作 | `packages/bruno-requests/src/cookies/index.ts` | Cookie Jar 封装、添加/获取 Cookie、过期检查 |
| Cookie 持久化存储 | `packages/bruno-electron/src/store/cookies.js` | 磁盘读写、加密解密、防抖写入 |

### 1.3 关键技术点

#### 1.3.1 内存 Cookie Jar 封装

在 `bruno-requests` 包中，`tough-cookie` 被封装为统一的 API 接口：

```typescript
// 核心 API 封装
const cookieJar = new CookieJar();

// 支持 callback 和 Promise 两种调用风格
cookieJarWrapper = {
  getCookie: (url, cookieName, callback?) => {...},
  hasCookie: (url, cookieName, callback?) => {...},
  getCookies: (url, callback?) => {...},
  setCookie: (url, nameOrCookieObj, valueOrCallback?) => {...},
  setCookies: (url, cookiesArray, callback?) => {...},
  clear: (callback?) => {...},
  deleteCookies: (url, callback?) => {...},
  deleteCookie: (url, cookieName, callback?) => {...}
}
```

#### 1.3.2 加密持久化策略

在 `bruno-electron` 的 `CookiesStore` 类中实现：

**加密机制**：
- 使用 AES 加密 Cookie 值
- Passkey 自身也加密存储于 `electron-store`
- 生成：`crypto.randomBytes(32).toString('hex')`

**防抖写入**：
```javascript
const DEBOUNCE_MS = 5000; // 5秒防抖

saveCookieJar(immediate = false) {
  if (immediate) {
    clearTimeout(this.#saveTimerId);
    return this.writeCookieJar();
  }
  // 延迟写入，避免频繁磁盘 IO
  this.#saveTimerId = setTimeout(() => {
    this.writeCookieJar();
  }, DEBOUNCE_MS);
}
```

**存储结构**：
```javascript
{
  encryptedPasskey: "加密的密钥",
  cookies: {
    "api.example.com": [
      {
        key: "session_id",
        value: "AES加密后的值",
        domain: ".example.com",
        path: "/",
        expires: "时间戳",
        ...
      }
    ]
  }
}
```

#### 1.3.3 Cookie 生命周期

1. **初始化加载**：应用启动时从磁盘读取并解密所有 Cookie，加载到内存 Jar
2. **响应保存**：从响应头 `Set-Cookie` 解析并添加到 Jar
3. **请求附加**：根据请求 URL 自动匹配有效 Cookie 并添加到请求头
4. **过期过滤**：获取 Cookie 时自动过滤已过期项
5. **持久化写入**：修改后延迟 5 秒写入磁盘（防抖）

---

## 二、OAuth2 多种 Grant Type 适配

### 2.1 支持的 Grant 类型

Bruno 完整支持 4 种标准 OAuth2 Grant Type：

| Grant Type | 典型场景 | 核心参数 |
|-----------|---------|---------|
| `client_credentials` | 服务端集成 | Client ID、Client Secret |
| `password` | 信任应用直接登录 | Username、Password、Client ID |
| `authorization_code` | Web 应用授权重定向 | Auth URL、Callback URL、PKCE |
| `implicit` | 纯前端应用简化流程 | Auth URL、Client ID |

### 2.2 核心实现模块

| 模块 | 文件路径 | 职责 |
|------|---------|------|
| Token 获取逻辑 | `packages/bruno-requests/src/auth/oauth2-helper.ts` | Token 请求、过期判断、额外参数处理 |
| Token 持久化（桌面） | `packages/bruno-electron/src/store/oauth2.js` | 按集合缓存、加密存储 Credentials |
| Token 存储（CLI） | `packages/bruno-cli/src/store/tokenStore.js` | 内存缓存实现 |
| Auth 类型定义 | `packages/bruno-schema-types/src/common/auth.ts` | TypeScript 类型定义 |

### 2.3 关键技术点

#### 2.3.1 Token 获取流程

```
调用 getOAuth2Token()
        │
        ▼
  检查 Token Store
        ├─ 存在且未过期 → 直接返回 Token
        ├─ 存在但已过期 → 清除旧 Token，继续获取
        └─ 不存在 → 发起新 Token 请求
                  │
                  ▼
        根据 Grant Type 分发：
        ├─ client_credentials → fetchTokenClientCredentials()
        ├─ password → fetchTokenPassword()
        └─ authorization_code/implicit → 浏览器授权流程
                  │
                  ▼
        Token 加密存入 Store
                  │
                  ▼
        返回 Access Token
```

#### 2.3.2 Credentials 放置策略

支持两种 Client 认证方式：

1. **Basic Auth Header**（默认）：
   ```
   Authorization: Basic base64(client_id:client_secret)
   ```

2. **Request Body**：
   ```
   grant_type=password&client_id=xxx&client_secret=xxx
   ```

#### 2.3.3 额外参数支持

支持在三个阶段注入自定义参数：
- `authorization`：授权请求阶段
- `token`：Token 请求阶段
- `refresh`：刷新 Token 阶段

每个参数可指定发送位置：
```typescript
type SendIn = 'headers' | 'queryparams' | 'body';

interface OAuthAdditionalParameter {
  name: string;
  value: string;
  enabled: boolean;
  sendIn: SendIn;
}
```

#### 2.3.4 过期判断与自动刷新

```typescript
const isTokenExpired = (credentials: any): boolean => {
  if (!credentials?.access_token) return true;
  if (!credentials?.expires_in || !credentials.created_at) return false;
  
  const expiryTime = credentials.created_at + credentials.expires_in * 1000;
  return Date.now() > expiryTime;
};
```

#### 2.3.5 持久化存储结构

**桌面端**（`Oauth2Store` 类）：
```javascript
{
  collections: [
    {
      collectionUid: "集合唯一标识",
      sessionId: "会话UUID（用于标识授权会话）",
      credentials: [
        {
          url: "https://token.endpoint/path",
          credentialsId: "标识ID",
          data: "AES加密的Token响应JSON"
        }
      ]
    }
  ]
}
```

**CLI 端**：内存 Map 结构，进程结束后丢失。

---

## 三、跨请求鉴权复用机制

### 3.1 鉴权方式总览

Bruno 支持 10 种鉴权方式，均实现跨请求复用：

| 鉴权方式 | 复用级别 | 实现方式 |
|---------|---------|---------|
| Basic Auth | 集合/请求 | Axios 内置 `auth` 配置 |
| Bearer Token | 集合/请求 | 直接设置 `Authorization` 头 |
| API Key | 集合/请求 | Header 或 Query 参数 |
| OAuth 1.0 | 集合/请求 | 签名算法 + 放置策略 |
| OAuth 2.0 | 集合/请求 | Token 存储 + 自动刷新 |
| AWS SigV4 | 集合/请求 | AWS 签名算法 |
| Digest Auth | 集合/请求 | 摘要认证 |
| NTLM | 集合/请求 | Windows 集成认证 |
| WSSE | 集合/请求 | WS-Security 头部 |
| Inherit | 文件夹层级 | 继承父级/集合配置 |

### 3.2 核心实现模块

| 模块 | 文件路径 | 职责 |
|------|---------|------|
| 请求准备 | `packages/bruno-electron/src/ipc/network/prepare-request.js` | 鉴权头设置、配置合并 |
| 集合合并 | `packages/bruno-electron/src/utils/collection.js` | `mergeAuth` 层级合并 |

### 3.3 关键技术点

#### 3.3.1 层级继承机制

鉴权配置支持 **集合 → 文件夹 → 请求** 三级继承：

```javascript
// setAuthHeaders 函数处理逻辑
if (collectionAuth && request.auth.mode === 'inherit') {
  // 使用集合级鉴权配置
  switch (collectionAuth.mode) {
    case 'basic':
      axiosRequest.basicAuth = { username, password };
      break;
    case 'bearer':
      axiosRequest.headers['Authorization'] = `Bearer ${token}`;
      break;
    // ... 其他鉴权方式
  }
}

// 请求级配置覆盖集合级
if (request.auth && request.auth.mode !== 'inherit') {
  // 应用请求专属鉴权
}
```

#### 3.3.2 Basic Auth 实现

直接使用 Axios 原生支持：
```javascript
axiosRequest.basicAuth = {
  username: get(request, 'auth.basic.username'),
  password: get(request, 'auth.basic.password')
};
```

#### 3.3.3 Bearer Token 实现

直接设置 Authorization 头：
```javascript
axiosRequest.headers['Authorization'] = `Bearer ${get(request, 'auth.bearer.token', '')}`;
```

#### 3.3.4 API Key 实现

支持两种放置位置：
```javascript
const apiKeyAuth = get(request, 'auth.apikey');

// 1. Header 方式
if (apiKeyAuth.placement === 'header') {
  axiosRequest.headers[apiKeyAuth.key] = apiKeyAuth.value;
  axiosRequest.apiKeyHeaderName = apiKeyAuth.key;
}

// 2. Query Params 方式（后续 URL 构建时处理）
else if (apiKeyAuth.placement === 'queryparams') {
  axiosRequest.apiKeyAuthValueForQueryParams = apiKeyAuth;
}
```

#### 3.3.5 OAuth2 Token 应用

获取 Token 后，根据 `tokenPlacement` 配置应用：

```javascript
// Token 放置策略
tokenPlacement: 'header' | 'queryparams',  // 放置位置
tokenHeaderPrefix: 'Bearer',               // Header 前缀
tokenQueryKey: 'access_token',             // Query 参数名
tokenSource: 'access_token' | 'id_token',  // 使用哪个 Token
```

应用时：
```javascript
// Header 方式
Authorization: `${tokenHeaderPrefix} ${accessToken}`

// Query Params 方式
?${tokenQueryKey}=${accessToken}
```

#### 3.3.6 WSSE 动态生成

WSSE 每次请求动态生成：
```javascript
const ts = new Date().toISOString();
const nonce = crypto.randomBytes(16).toString('hex');

// SHA1 摘要
const hash = crypto.createHash('sha1');
hash.update(nonce + ts + password);
const digest = Buffer.from(hash.digest('hex')).toString('base64');

axiosRequest.headers['X-WSSE'] = 
  `UsernameToken Username="${username}", PasswordDigest="${digest}", Nonce="${nonce}", Created="${ts}"`;
```

### 3.4 跨请求复用的核心保障

1. **配置持久化**：所有鉴权配置随集合文件存储，重启不丢失
2. **Token 缓存**：OAuth2 Token 单独加密存储，跨请求共享
3. **层级继承**：集合级配置一次设置，所有请求默认继承
4. **环境变量支持**：敏感信息通过环境变量注入，实现跨环境复用
5. **Credentials ID 隔离**：同一 Token URL 可有多组独立 Credentials

---

## 四、架构总结

### 4.1 整体数据流

```
┌─────────────────────────────────────────────────────────┐
│                    集合配置文件                           │
│  { collection: { request: { auth: {...} } } }           │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│                   prepareRequest()                       │
│  - mergeAuth 层级合并                                   │
│  - setAuthHeaders 应用鉴权配置                           │
└──────────────────────┬──────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│  Basic      │ │  Bearer    │ │  API Key    │
│  Auth       │ │  Token     │ │  Header/Query│
└─────────────┘ └─────────────┘ └─────────────┘
                       │
                       ▼
              ┌──────────────────┐
              │  OAuth2 Helper   │
              │  - getOAuth2Token│
              │  - 过期检查       │
              └────────┬─────────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ Oauth2Store │ │ tokenStore  │ │ Cookie Jar  │
│ (Electron)  │ │ (CLI)       │ │ (内存+磁盘)  │
└─────────────┘ └─────────────┘ └─────────────┘
```

### 4.2 设计亮点

1. **分层清晰**：鉴权逻辑与存储层分离，便于适配不同运行环境（Electron/CLI/Web）
2. **安全可靠**：敏感数据全部 AES 加密落盘，内存仅临时持有
3. **扩展性强**：新增鉴权方式只需在 `setAuthHeaders` 添加分支
4. **用户友好**：层级继承减少重复配置，Token 自动刷新提升体验
5. **性能优化**：Cookie 写入防抖、Token 内存缓存，避免不必要 IO

### 4.3 关键文件索引

| 功能 | 文件路径 |
|------|---------|
| Cookie 核心 | `packages/bruno-requests/src/cookies/index.ts` |
| Cookie 存储 | `packages/bruno-electron/src/store/cookies.js` |
| OAuth2 核心 | `packages/bruno-requests/src/auth/oauth2-helper.ts` |
| OAuth2 存储 | `packages/bruno-electron/src/store/oauth2.js` |
| 请求准备 | `packages/bruno-electron/src/ipc/network/prepare-request.js` |
| Auth 类型 | `packages/bruno-schema-types/src/common/auth.ts` |
| CLI Token 存储 | `packages/bruno-cli/src/store/tokenStore.js` |
