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

### 2.1 支持的 Grant 类型与分流架构

Bruno 完整支持 **4 种标准 OAuth2 Grant Type**，采用 **两层架构** 处理不同授权模式：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          configureRequest()                              │
│                  packages/bruno-electron/src/ipc/network/index.js       │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
          ┌─────────────────────────┼─────────────────────────┐
          │                         │                         │
          ▼                         ▼                         ▼
┌───────────────────┐    ┌───────────────────┐    ┌───────────────────┐
│ authorization_code│    │     implicit      │    │ client_credentials │
│  + password       │    │                   │    │                   │
└─────────┬─────────┘    └─────────┬─────────┘    └─────────┬─────────┘
          │                        │                        │
          ▼                        ▼                        ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                             授权层分层架构                                 │
├───────────────────────────────────────────────────────────────────────────┤
│  🔴 Layer 1/2: 浏览器交互模式（需用户参与）                                │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ authorizeUserInWindow()  │  authorizeUserInSystemBrowser()           │ │
│  │  应用内浏览器窗口        │  系统默认浏览器                            │ │
│  └──────────────────────────┴───────────────────────────────────────────┘ │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │  authorization_code: 获取 code → 回调 URL → POST token URL           │ │
│  │  implicit: 直接从 URL hash 片段获取 access_token                     │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  🟢 Layer 3: 纯后端 API 模式（无用户交互）                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │  getCredentialsFromTokenUrl() → POST access_token_url 直接获取 Token │ │
│  │  ├─ client_credentials: client_id + client_secret                    │ │
│  │  └─ password: username + password + client_id (+ client_secret)      │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────────┘
```

### 2.2 四种 Grant Type 准确分流关系

| Grant Type | 所属层级 | 调用入口函数 | 关键流程 | 典型场景 |
|-----------|---------|------------|---------|---------|
| **authorization_code** | 🔴 Layer 1/2 | `getOAuth2TokenUsingAuthorizationCode()` | 浏览器弹窗授权 → 获取 code → POST token URL → 获取 access_token | Web 应用、第三方登录 |
| **implicit** | 🔴 Layer 1/2 | `getOAuth2TokenUsingImplicitGrant()` | 浏览器弹窗授权 → 直接从 URL hash 获取 token | 纯前端 SPA 应用 |
| **client_credentials** | 🟢 Layer 3 | `getOAuth2TokenUsingClientCredentials()` | 直接 POST token URL，无浏览器交互 | 服务间集成、M2M |
| **password** | 🟢 Layer 3 | `getOAuth2TokenUsingPasswordCredentials()` | 直接 POST token URL，无浏览器交互 | 可信内部应用 |

### 2.3 核心实现模块

| 模块 | 文件路径 | 职责 |
|------|---------|------|
| **主调用入口** | `packages/bruno-electron/src/ipc/network/index.js` | `configureRequest()` 中根据 grantType 分流 |
| Token 获取逻辑 | `packages/bruno-electron/src/utils/oauth2.js` | 4 个 Grant Type 独立函数、Token 缓存检查、自动刷新 |
| 应用内浏览器授权 | `packages/bruno-electron/src/ipc/network/authorizeUserInWindow.js` | Electron BrowserWindow 弹窗、URL 拦截、Code/Token 提取 |
| 系统浏览器授权 | `packages/bruno-electron/src/ipc/network/authorizeUserInSystemBrowser.js` | 系统默认浏览器打开、自定义协议回调 |
| Token 持久化 | `packages/bruno-electron/src/store/oauth2.js` | 按集合缓存、加密存储 Credentials |
| CLI Token 存储 | `packages/bruno-cli/src/store/tokenStore.js` | 内存缓存实现 |
| Auth 类型定义 | `packages/bruno-schema-types/src/common/auth.ts` | TypeScript 类型定义 |

### 2.4 关键技术点

#### 2.4.1 实际调用链（证据）

在 `packages/bruno-electron/src/ipc/network/index.js` 第 228-297 行：

```javascript
switch (grantType) {
  case 'authorization_code':
    interpolateVars(requestCopy, envVars, runtimeVariables, processEnvVars, promptVariables);
    ({ credentials, url: oauth2Url, credentialsId, debugInfo } = 
      await getOAuth2TokenUsingAuthorizationCode({ 
        request: requestCopy, 
        collectionUid, 
        certsAndProxyConfigForTokenUrl, 
        certsAndProxyConfigForRefreshUrl 
      }));
    break;
    
  case 'implicit':
    interpolateVars(requestCopy, envVars, runtimeVariables, processEnvVars, promptVariables);
    ({ credentials, url: oauth2Url, credentialsId, debugInfo } = 
      await getOAuth2TokenUsingImplicitGrant({ 
        request: requestCopy, 
        collectionUid 
      }));
    break;
    
  case 'client_credentials':
    interpolateVars(requestCopy, envVars, runtimeVariables, processEnvVars, promptVariables);
    ({ credentials, url: oauth2Url, credentialsId, debugInfo } = 
      await getOAuth2TokenUsingClientCredentials({ 
        request: requestCopy, 
        collectionUid, 
        certsAndProxyConfigForTokenUrl, 
        certsAndProxyConfigForRefreshUrl 
      }));
    break;
    
  case 'password':
    interpolateVars(requestCopy, envVars, runtimeVariables, processEnvVars, promptVariables);
    ({ credentials, url: oauth2Url, credentialsId, debugInfo } = 
      await getOAuth2TokenUsingPasswordCredentials({ 
        request: requestCopy, 
        collectionUid, 
        certsAndProxyConfigForTokenUrl, 
        certsAndProxyConfigForRefreshUrl 
      }));
    break;
}
```

**重要修正**：不存在统一的 `getOAuth2Token()` 入口函数。四种 Grant Type 从 `configureRequest()` 直接并行分流，各自走独立实现。

#### 2.4.2 浏览器授权窗口实现（证据）

在 `packages/bruno-electron/src/ipc/network/authorizeUserInWindow.js` 中：

```javascript
const authorizeUserInWindow = ({ authorizeUrl, callbackUrl, session, additionalHeaders = {}, grantType = 'authorization_code' }) => {
  // ...
  // 拦截 URL 变化，检测回调
  function onWindowRedirect(url) {
    // 先处理错误响应
    if (urlObj.searchParams.has('error')) {
      const error = urlObj.searchParams.get('error');
      reject(new Error(JSON.stringify(errorData)));
      window.close();
      return;
    }
    
    // 匹配回调 URL：需要有 code 参数 或 hash 片段
    if (callbackUrlObj && matchesCallbackUrl(urlObj, callbackUrlObj)) {
      finalUrl = url;
      window.close();
      return;
    }
  }
  
  // 窗口关闭时提取结果
  window.on('close', () => {
    if (finalUrl) {
      if (grantType === 'implicit') {
        // implicit flow: 从 hash 片段提取 token
        const urlWithHash = new URL(finalUrl);
        const hash = urlWithHash.hash.substring(1);
        const hashParams = new URLSearchParams(hash);
        const implicitTokens = {
          access_token: hashParams.get('access_token'),
          token_type: hashParams.get('token_type'),
          expires_in: hashParams.get('expires_in'),
          state: hashParams.get('state'),
          scope: hashParams.get('scope')
        };
        return resolve({ implicitTokens, debugInfo });
      } else {
        // authorization_code flow: 从 query params 提取 code
        const callbackUrlWithCode = new URL(finalUrl);
        const authorizationCode = callbackUrlWithCode.searchParams.get('code');
        return resolve({ authorizationCode, debugInfo });
      }
    }
  });
};
```

#### 2.4.3 Token 缓存检查与自动刷新（证据）

所有 4 个 Grant Type 函数都共享相同的缓存检查逻辑（以 `getOAuth2TokenUsingAuthorizationCode` 为例）：

```javascript
if (!forceFetch) {
  const storedCredentials = getStoredOauth2Credentials({ collectionUid, url, credentialsId });
  
  if (storedCredentials) {
    if (!isTokenExpired(storedCredentials)) {
      // Token 有效，直接返回缓存
      return { collectionUid, url, credentials: storedCredentials, credentialsId };
    } else {
      // Token 过期
      if (autoRefreshToken && storedCredentials.refresh_token) {
        // 自动刷新 Token
        try {
          const refreshedCredentialsData = await refreshOauth2Token({ 
            requestCopy, 
            collectionUid, 
            certsAndProxyConfig: certsAndProxyConfigForRefreshUrl 
          });
          return { collectionUid, url, credentials: refreshedCredentialsData.credentials, credentialsId };
        } catch (error) {
          // 刷新失败，清除缓存后重新获取
          clearOauth2Credentials({ collectionUid, url, credentialsId });
          if (autoFetchToken) { /* 重新获取 */ }
        }
      }
    }
  }
}
```

**注意**：`implicit` 模式不支持 refresh token，因为 OAuth2 规范不允许。

#### 2.4.4 PKCE 支持（授权码模式）

```javascript
const getOAuth2TokenUsingAuthorizationCode = async (...) => {
  let codeVerifier = generateCodeVerifier();
  let codeChallenge = generateCodeChallenge(codeVerifier);
  
  // 构建授权 URL 时添加 PKCE 参数
  authorizationUrlWithQueryParams.searchParams.append('code_challenge', codeChallenge);
  authorizationUrlWithQueryParams.searchParams.append('code_challenge_method', 'S256');
  
  // 换取 Token 时提交 code_verifier
  if (pkce) {
    data['code_verifier'] = codeVerifier;
  }
};

// PKCE 辅助函数
const generateCodeVerifier = () => crypto.randomBytes(22).toString('hex');
const generateCodeChallenge = (codeVerifier) => {
  const hash = crypto.createHash('sha256');
  hash.update(codeVerifier);
  return hash.digest('base64').replace(/\+/g, '-').replace(/\//g, '_').replace(/=/g, '');
};
```

#### 2.4.5 Credentials 放置策略

支持两种 Client 认证方式：

1. **Basic Auth Header**（默认）：
   ```
   Authorization: Basic base64(client_id:client_secret)
   ```

2. **Request Body**：
   ```
   grant_type=password&client_id=xxx&client_secret=xxx
   ```

#### 2.4.6 额外参数支持

支持在三个阶段注入自定义参数：
- `authorization`：授权请求阶段（仅适用于 authorization_code 和 implicit）
- `token`：Token 请求阶段（所有 grant type）
- `refresh`：刷新 Token 阶段（authorization_code、client_credentials、password）

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

#### 2.4.7 持久化存储结构

**桌面端**（`Oauth2Store` 类）：
```javascript
{
  collections: [
    {
      collectionUid: "集合唯一标识",
      sessionId: "会话UUID（用于隔离浏览器 session）",
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
| WSSE | 集合/请求 | WS-Security 头部（动态生成 nonce） |
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

WSSE 每次请求动态生成 nonce 和 timestamp：
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
2. **Token 缓存**：OAuth2 Token 单独加密存储，跨请求共享，自动过期检查
3. **层级继承**：集合级配置一次设置，所有请求默认继承，减少重复配置
4. **环境变量支持**：敏感信息通过环境变量注入，实现跨环境复用
5. **Credentials ID 隔离**：同一 Token URL 可有多组独立 Credentials，支持多账号场景
6. **Session 隔离**：OAuth2 浏览器授权使用独立 session partition，避免集合间 Cookie 污染

---

## 四、架构总结

### 4.1 整体数据流

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              集合配置文件                                  │
│  { collection: { request: { auth: {...} } } }                            │
└─────────────────────────────────────┬─────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                            prepareRequest()                               │
│  - mergeAuth 层级合并                                                    │
│  - setAuthHeaders 应用鉴权配置                                            │
└─────────────────────────────────────┬─────────────────────────────────────┘
                                      │
          ┌───────────────────────────┼───────────────────────────┐
          ▼                           ▼                           ▼
┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
│   Basic Auth    │       │  Bearer Token   │       │    API Key      │
│                 │       │                 │       │  Header/Query   │
└─────────────────┘       └─────────────────┘       └─────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          configureRequest()                               │
│  ├─ OAuth2 Grant Type 分流                                               │
│  ├─ 🔴 authorization_code / implicit → 浏览器授权窗口                     │
│  ├─ 🟢 client_credentials / password → 直接 API 调用                      │
│  └─ Token 过期检查 → 自动刷新 / 重新获取                                   │
└─────────────────────────────────────┬─────────────────────────────────────┘
                                      │
          ┌───────────────────────────┼───────────────────────────┐
          ▼                           ▼                           ▼
┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
│  Oauth2Store    │       │  tokenStore CLI │       │  Cookie Jar     │
│  (Electron)     │       │   (内存)        │       │  (内存+磁盘)    │
└─────────────────┘       └─────────────────┘       └─────────────────┘
```

### 4.2 设计亮点

1. **分层清晰**：鉴权逻辑与存储层分离，便于适配不同运行环境（Electron/CLI/Web）
2. **安全可靠**：敏感数据全部 AES 加密落盘，内存仅临时持有；OAuth2 浏览器授权使用独立 session 隔离
3. **扩展性强**：新增鉴权方式只需在 `setAuthHeaders` 添加分支；新增 Grant Type 只需添加独立函数
4. **用户友好**：层级继承减少重复配置，Token 自动刷新提升体验；双浏览器授权模式适应不同场景
5. **性能优化**：Cookie 写入防抖、Token 内存缓存 + 过期检查，避免不必要 IO 和网络请求
6. **调试友好**：所有 OAuth2 流程保留完整的 request/response debugInfo，便于排查授权问题

### 4.3 关键文件索引

| 功能 | 文件路径 |
|------|---------|
| Cookie 核心 | `packages/bruno-requests/src/cookies/index.ts` |
| Cookie 存储 | `packages/bruno-electron/src/store/cookies.js` |
| **OAuth2 主入口** | `packages/bruno-electron/src/ipc/network/index.js` (configureRequest) |
| **OAuth2 四种 Grant 实现** | `packages/bruno-electron/src/utils/oauth2.js` |
| **应用内浏览器授权** | `packages/bruno-electron/src/ipc/network/authorizeUserInWindow.js` |
| **系统浏览器授权** | `packages/bruno-electron/src/ipc/network/authorizeUserInSystemBrowser.js` |
| OAuth2 存储 | `packages/bruno-electron/src/store/oauth2.js` |
| 请求准备 | `packages/bruno-electron/src/ipc/network/prepare-request.js` |
| Auth 类型 | `packages/bruno-schema-types/src/common/auth.ts` |
| CLI Token 存储 | `packages/bruno-cli/src/store/tokenStore.js` |

---

## 五、修正说明

本报告 v2 版本相对于初版的关键修正：

1. **✓ 删除了不存在的 `getOAuth2Token()` 统一入口描述**
2. **✓ 补充了四种 Grant Type 的准确分流关系与调用入口函数**
3. **✓ 新增授权模式分层架构图（Layer 1/2/3）**
4. **✓ 补充了各关键节点的源码证据与行号引用**
5. **✓ 补充了 implicit flow 不支持 refresh token 的说明**
6. **✓ 新增 Session 隔离机制的说明**
7. **✓ 补充了 WSSE 动态生成 nonce 的跨请求复用说明**
