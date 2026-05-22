# Bruno 桌面客户端 TLS 证书装载与代理设置分析

## 一、请求执行链整体架构

### 1.1 请求执行流程图

```
用户发起请求
    ↓
IPC: send-http-request (bruno-electron/src/ipc/network/index.js:1205)
    ↓
runRequest() (bruno-electron/src/ipc/network/index.js:737)
    ├─ prepareRequest() - 准备请求对象
    ├─ buildCertsAndProxyConfig() - 为 bru.sendRequest 构建配置
    ├─ runPreRequest() - 执行前置脚本
    │   └─ interpolateVars() - 变量插值（证书/代理配置中的变量也会被插值）
    └─ configureRequest() (bruno-electron/src/ipc/network/index.js:101)
        ├─ getCertsAndProxyConfig() (bruno-electron/src/ipc/network/cert-utils.js:13)
        │   ├─ getCACertificates() - 加载 CA 证书 (bruno-requests/src/utils/ca-cert.ts:92)
        │   ├─ 客户端证书域名匹配与加载
        │   └─ 代理配置层级解析
        └─ makeAxiosInstance() (bruno-electron/src/ipc/network/axios-instance.js:74)
            └─ axios instance with interceptors
                └─ request interceptor
                    └─ setupProxyAgents() (bruno-electron/src/utils/proxy-util.js)
                        ├─ shouldUseProxy() - 代理绕过检测
                        └─ 创建代理 Agent (HttpProxyAgent / HttpsProxyAgent / SocksProxyAgent)
    ↓
axiosInstance(request) - 实际发送请求
    ↓
response interceptor - 处理响应、重定向
    ↓
重定向时重新调用 setupProxyAgents() - 为重定向 URL 重新评估代理和证书
```

### 1.2 关键接入点位置

| 阶段 | 文件位置 | 关键函数 | 作用 |
|------|---------|---------|------|
| 配置准备 | `bruno-electron/src/ipc/network/cert-utils.js:13` | `getCertsAndProxyConfig()` | 集中获取证书和代理配置 |
| 配置准备 | `bruno-electron/src/ipc/network/cert-utils.js:171` | `buildCertsAndProxyConfig()` | 为 `bru.sendRequest()` 构建配置 |
| CA 证书加载 | `bruno-requests/src/utils/ca-cert.ts:92` | `getCACertificates()` | 加载并合并各类 CA 证书 |
| Agent 创建 | `bruno-requests/src/utils/http-https-agents.ts:530` | `getHttpHttpsAgents()` | 为请求库创建 HTTP/HTTPS Agent（供 CLI 和脚本使用） |
| Agent 创建 | `bruno-electron/src/utils/proxy-util.js:105` | `setupProxyAgents()` | Electron 端代理 Agent 创建 |
| 请求拦截 | `bruno-electron/src/ipc/network/axios-instance.js:108` | `instance.interceptors.request` | 请求发送前最终注入代理和证书 |
| 重定向处理 | `bruno-electron/src/ipc/network/axios-instance.js:420` | 重定向拦截器中 | 重定向时重新评估代理和证书配置 |
| 脚本请求 | `bruno-requests/src/scripting/send-request.ts:22` | `createSendRequest()` | 脚本中 `bru.sendRequest()` 的证书和代理继承 |

---

## 二、自定义 CA 证书装载流程

### 2.1 CA 证书来源与优先级

`getCACertificates()` 函数 (`bruno-requests/src/utils/ca-cert.ts:92`) 负责聚合四类证书：

| 证书类型 | 来源 | 优先级说明 |
|---------|------|-----------|
| 系统 CA 证书 | `tls.getCACertificates('system')` | 从操作系统证书库获取 |
| Node.js 根证书 | `tls.rootCertificates` | Node.js 内置根证书 |
| 自定义 CA 证书 | 用户指定的证书文件路径 | 由 `caCertFilePath` 参数指定 |
| NODE_EXTRA_CA_CERTS | 环境变量指定的文件 | 从 `process.env.NODE_EXTRA_CA_CERTS` 读取 |

### 2.2 组合策略

```typescript
// ca-cert.ts:92-160
const getCACertificates = ({ caCertFilePath, shouldKeepDefaultCerts = true }) => {
  // 情况1: 指定了自定义 CA 证书文件
  if (caCertFilePath) {
    // 加载自定义证书
    customCerts = [readFile(caCertFilePath)]
    
    if (shouldKeepDefaultCerts) {
      // 保留默认证书：自定义 + 系统 + Node根 + NODE_EXTRA
      systemCerts = getSystemCerts()
      rootCerts = [...tls.rootCertificates]
    }
    // 否则：仅使用自定义 + NODE_EXTRA（忽略系统和Node根证书）
  } 
  // 情况2: 未指定自定义 CA
  else {
    // 使用：系统 + Node根 + NODE_EXTRA
    systemCerts = getSystemCerts()
    rootCerts = [...tls.rootCertificates]
  }
  
  // 始终加载 NODE_EXTRA_CA_CERTS
  nodeExtraCerts = getNodeExtraCACerts()
  
  // 合并去重（使用 Set，按顺序添加）
  return mergeCA(systemCerts, rootCerts, customCerts, nodeExtraCerts)
}
```

**关键设计决策：**
- `shouldKeepDefaultCerts` 默认 `true`：自定义证书追加到默认证书链
- 设为 `false` 时：仅信任自定义证书，实现严格的证书锁定
- `NODE_EXTRA_CA_CERTS` 始终被加载，不受 `shouldKeepDefaultCerts` 影响
- 使用 `Set` 去重，避免重复证书

### 2.3 接入条件

CA 证书仅在 TLS 验证启用时加载：

```javascript
// cert-utils.js:36
if (preferencesUtil.shouldVerifyTls()) {
  const caCertificatesData = getCACertificates({...});
  httpsAgentRequestFields['ca'] = caCertificatesData.caCertificates;
}

// http-https-agents.ts:254
if (options.shouldVerifyTls) {
  const caCertificatesData = getCACertificates({...});
  certsConfig.ca = caCertificatesData.caCertificates;
}
```

如果 `shouldVerifyTls` 为 `false`，则设置 `rejectUnauthorized: false`，完全跳过证书验证。

---

## 三、客户端证书装载流程

### 3.1 配置结构

客户端证书配置存储在 `bruno.json` 中：

```javascript
// cert-utils.js:69
const clientCertConfig = get(brunoConfig, 'clientCertificates.certs', []);
```

单个客户端证书配置项：
```typescript
type ClientCertificate = {
  domain: string;           // 匹配的域名，支持通配符
  type: 'cert' | 'pfx';     // 证书类型
  certFilePath?: string;    // cert 类型：证书文件路径
  keyFilePath?: string;     // cert 类型：私钥文件路径
  pfxFilePath?: string;     // pfx 类型：PKCS#12 文件路径
  passphrase?: string;      // 私钥或 PFX 的密码
};
```

### 3.2 域名匹配逻辑

```javascript
// cert-utils.js:71-106
for (let clientCert of clientCertConfig) {
  const domain = interpolateString(clientCert?.domain, interpolationOptions);
  if (domain) {
    // 构建正则：支持 https://、grpc://、grpcs://、ws://、wss:// 前缀
    // 支持通配符 *，例如 *.example.com
    const hostRegex = '^(https:\\/\\/|grpc:\\/\\/|grpcs:\\/\\/|ws:\\/\\/|wss:\\/\\/)?'
      + domain.replaceAll('.', '\\.').replaceAll('*', '.*');
    
    const requestUrl = interpolateString(request.url, interpolationOptions);
    if (requestUrl && requestUrl.match(hostRegex)) {
      // 匹配成功，加载证书
      if (type === 'cert') {
        httpsAgentRequestFields['cert'] = fs.readFileSync(certFilePath);
        httpsAgentRequestFields['key'] = fs.readFileSync(keyFilePath);
      } else if (type === 'pfx') {
        httpsAgentRequestFields['pfx'] = fs.readFileSync(pfxFilePath);
      }
      httpsAgentRequestFields['passphrase'] = interpolateString(clientCert.passphrase, ...);
      
      break;  // 找到第一个匹配即停止
    }
  }
}
```

**关键特性：**
1. **变量插值**：所有字段（domain、certFilePath、keyFilePath、pfxFilePath、passphrase）都支持变量插值
2. **路径解析**：相对路径相对于 collection 根目录解析
3. **匹配顺序**：按配置顺序匹配，第一个匹配的证书生效
4. **协议前缀**：自动匹配多种协议前缀（http/https/grpc/grpcs/ws/wss）
5. **通配符支持**：`*.example.com` 匹配所有子域名

### 3.3 CLI 端实现（http-https-agents.ts）

```typescript
// http-https-agents.ts:266-311
const clientCertConfig = get(clientCertificates, 'certs', []) as ClientCertificate[];
for (const clientCert of clientCertConfig) {
  const domain = clientCert?.domain;
  const hostRegex = '^(https:\\/\\/|grpc:\\/\\/|grpcs:\\/\\/)?' 
    + domain.replace(/\./g, '\\.').replace(/\*/g, '.*');
  
  if (requestUrl && requestUrl.match(hostRegex)) {
    // 加载证书...
    break;
  }
}
```

注意：CLI 端的正则匹配不包含 `ws://` 和 `wss://` 前缀（与 Electron 端差异）。

---

## 四、代理设置层级与优先级

### 4.1 代理配置层级

代理配置遵循"集合级 → 应用级 → 系统级"的优先级链：

```
┌─────────────────────────────────────────────────────────┐
│                代理配置优先级（从高到低）                │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. 请求级 noproxy 标记                                 │
│     └─ options.noproxy = true 时，完全禁用代理          │
│                                                         │
│  2. 集合级代理 (bruno.json)                             │
│     ├─ disabled: true    → 禁用代理                     │
│     ├─ inherit: false    → 使用集合自身配置             │
│     └─ inherit: true     → 继承应用级配置               │
│                                                         │
│  3. 应用级代理 (全局偏好设置)                           │
│     ├─ disabled: true       → 禁用代理                  │
│     ├─ source: 'pac'        → 使用 PAC 自动配置         │
│     ├─ source: 'inherit'    → 继承系统代理              │
│     └─ source: 'manual'     → 使用手动配置              │
│                                                         │
│  4. 系统级代理 (环境变量)                               │
│     ├─ http_proxy / HTTP_PROXY                          │
│     ├─ https_proxy / HTTPS_PROXY                        │
│     └─ no_proxy / NO_PROXY                              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 4.2 集合级代理配置解析

```javascript
// cert-utils.js:126-162
const collectionProxyConfig = get(brunoConfig, 'proxy', {});
const collectionProxyDisabled = get(collectionProxyConfig, 'disabled', false);
const collectionProxyInherit = get(collectionProxyConfig, 'inherit', true);

if (!collectionProxyDisabled && !collectionProxyInherit) {
  // 优先级最高：使用集合自身的代理配置
  proxyConfig = collectionProxyConfigData;
  proxyMode = 'on';
} else if (!collectionProxyDisabled && collectionProxyInherit) {
  // 继承应用级配置
  const globalProxy = preferencesUtil.getGlobalProxyConfig();
  if (!globalDisabled) {
    if (globalProxySource === 'pac') {
      proxyMode = 'pac';
    } else if (globalProxySource === 'inherit') {
      proxyMode = 'system';  // 继承系统代理
    } else {  // manual
      proxyConfig = globalProxyConfigData;
      proxyMode = 'on';
    }
  }
} else {
  // 集合级禁用代理
  proxyModeReason = 'Collection-level proxy is disabled';
}
```

### 4.3 代理模式说明

| 模式 | 说明 |
|------|------|
| `off` | 不使用代理，直接连接 |
| `on` | 使用手动配置的代理（集合级或应用级） |
| `system` | 使用系统环境变量配置的代理 |
| `pac` | 通过 PAC（Proxy Auto-Configuration）脚本自动选择代理 |

### 4.4 旧格式兼容（proxy-util.ts）

支持新旧两种配置格式的自动转换：

```typescript
// 旧格式
{
  enabled: true | false | 'global',
  protocol: 'http',
  hostname: 'proxy.example.com',
  port: 8080,
  auth: { enabled: true, username: 'user', password: 'pass' }
}

// 转换为新格式
{
  disabled: false,      // enabled: true → disabled: false
  inherit: false,       // enabled: true → inherit: false
  config: {
    protocol: 'http',
    hostname: 'proxy.example.com',
    port: 8080,
    auth: { disabled: false, username: 'user', password: 'pass' }
  }
}
```

转换规则：
- `enabled: true` → `disabled: false, inherit: false`
- `enabled: false` → `disabled: true, inherit: false`
- `enabled: 'global'` → `disabled: false, inherit: true`
- `auth.enabled: false` → `auth.disabled: true`

### 4.5 代理绕过规则

```javascript
// http-https-agents.ts:151-199
const shouldUseProxy = (url, proxyBypass) => {
  if (proxyBypass === '*') return false;  // 全部绕过
  
  // proxyBypass 支持逗号/分号/空格分隔的多个主机
  return proxyBypass.split(/[,;\s]/).every(function (dontProxyFor) {
    // 支持精确匹配：example.com
    // 支持通配符：*.example.com
    // 支持端口匹配：example.com:8080
  });
};
```

绕过规则支持：
- `*` - 绕过所有域名
- `example.com` - 精确匹配
- `*.example.com` - 通配符匹配所有子域名
- `example.com:8080` - 指定端口
- 多个规则用逗号、分号或空格分隔

---

## 五、代理凭据与 Agent 创建

### 5.1 代理 URI 构建

```javascript
// http-https-agents.ts:430-436
if (proxyAuthEnabled) {
  const proxyAuthUsername = encodeURIComponent(get(proxyConfig, 'auth.username', ''));
  const proxyAuthPassword = encodeURIComponent(get(proxyConfig, 'auth.password', ''));
  proxyUri = `${proxyProtocol}://${proxyAuthUsername}:${proxyAuthPassword}@${proxyHostname}${uriPort}`;
} else {
  proxyUri = `${proxyProtocol}://${proxyHostname}${uriPort}`;
}
```

凭据直接编码到代理 URI 中，支持 HTTP Basic Auth。

### 5.2 Agent 类型选择

```typescript
// http-https-agents.ts:444-456
if (socksEnabled) {
  if (isHttpsRequest) {
    httpsAgent = new SocksProxyAgent(proxyUri, tlsOptions);
  } else {
    httpAgent = new SocksProxyAgent(proxyUri, httpProxyAgentOptions);
  }
} else {
  if (isHttpsRequest) {
    httpsAgent = new PatchedHttpsProxyAgent(proxyUri, tlsOptions);
  } else {
    httpAgent = new HttpProxyAgent(proxyUri, httpProxyAgentOptions);
  }
}
```

| 代理类型 | HTTP 请求 | HTTPS 请求 |
|---------|----------|-----------|
| HTTP 代理 | `HttpProxyAgent` | `PatchedHttpsProxyAgent` |
| SOCKS 代理 | `SocksProxyAgent` | `SocksProxyAgent` |

### 5.3 PatchedHttpsProxyAgent - TLS 选项修复

**问题**：上游 `HttpsProxyAgent` 在将隧道 socket 升级为 TLS 时，会忽略构造时传入的 TLS 选项（如客户端证书、自定义 CA 等），导致这些选项只应用于代理连接，不应用于目标服务器连接。

**解决方案** (`http-https-agents.ts:218-240`)：

```typescript
class PatchedHttpsProxyAgent extends HttpsProxyAgent<any> {
  private constructorOpts: any;

  constructor(proxy: string, opts: any) {
    super(proxy, opts);
    this.constructorOpts = opts;  // 保存构造选项
  }

  async connect(req: any, opts: any) {
    const targetOpts = { ...opts };
    
    // 在升级到目标服务器 TLS 时，转发关键 TLS 选项
    const TARGET_TLS_OPTIONS = ['cert', 'key', 'pfx', 'passphrase', 'rejectUnauthorized', 'secureContext'];
    for (const key of TARGET_TLS_OPTIONS) {
      if (key in this.constructorOpts) {
        targetOpts[key] = this.constructorOpts[key];
      }
    }

    return super.connect(req, targetOpts);
  }
}
```

**转发的 TLS 选项**：
- `cert`, `key` - 客户端证书和私钥
- `pfx`, `passphrase` - PKCS#12 证书和密码
- `rejectUnauthorized` - 是否验证服务器证书
- `secureContext` - 预构建的安全上下文（包含 CA 证书）

### 5.4 HTTPS 代理的特殊处理

当代理本身使用 HTTPS 协议时，即使目标是 HTTP 请求，连接代理也需要 TLS：

```typescript
// http-https-agents.ts:439-441
const isHttpsProxy = proxyProtocol === 'https';
const httpProxyAgentOptions = isHttpsProxy ? { keepAlive: true, ...tlsOptions } : { keepAlive: true };
```

此时 `tlsOptions`（包含 CA 证书、验证设置等）会应用于代理连接本身。

---

## 六、PAC 代理支持

### 6.1 PAC 解析流程

```typescript
// http-https-agents.ts:458-491
} else if (proxyMode === 'pac') {
  const pacSource = get(proxyConfig, 'pac.source');
  if (pacSource && requestUrl) {
    try {
      // 创建 PAC 解析器，传入 TLS 选项
      const resolver = await getPacResolver({ 
        pacSource, 
        httpsAgentRequestFields: { 
          ca: tlsOptions.ca, 
          rejectUnauthorized: tlsOptions.rejectUnauthorized,
          minVersion: tlsOptions.minVersion 
        }
      });
      
      // 对当前 URL 执行 PAC 脚本
      const directives = await resolver.resolve(requestUrl);
      
      // 解析返回的代理指令
      if (directives && directives.length) {
        const first = directives[0];
        if (/^(PROXY|HTTPS?)\s+/i.test(first)) {
          // PROXY host:port 或 HTTPS host:port
          // 创建 HttpsProxyAgent / HttpProxyAgent
        } else if (/^SOCKS/i.test(first)) {
          // SOCKS host:port 或 SOCKS4 host:port
          // 创建 SocksProxyAgent
        }
      }
    } catch {
      // PAC 解析失败，回退到直接连接
    }
  }
}
```

### 6.2 PAC 返回值处理

| PAC 返回指令 | 处理方式 |
|-------------|---------|
| `PROXY host:port` | 使用 HTTP 代理 |
| `HTTP host:port` | 使用 HTTP 代理 |
| `HTTPS host:port` | 使用 HTTPS 代理 |
| `SOCKS host:port` | 使用 SOCKS5 代理 |
| `SOCKS4 host:port` | 使用 SOCKS4 代理 |
| `DIRECT` | 直接连接（不使用代理） |

多个返回值按顺序尝试，第一个可用的生效。

---

## 七、三者组合策略与交互关系

### 7.1 配置聚合流程

```
每个请求执行前：
    ↓
1.  getCertsAndProxyConfig() 集中获取所有配置
    ├─ CA 证书配置（依赖 shouldVerifyTls）
    ├─ 客户端证书（依赖域名匹配）
    └─ 代理配置（依赖层级优先级）
    ↓
2.  配置注入 httpsAgentRequestFields
    ├─ ca: 合并后的 CA 证书链
    ├─ cert, key / pfx: 匹配的客户端证书
    ├─ passphrase: 证书密码
    └─ rejectUnauthorized: TLS 验证开关
    ↓
3.  makeAxiosInstance() 创建实例并传入配置
    ↓
4.  请求拦截器中调用 setupProxyAgents()
    ├─ 评估代理绕过规则
    ├─ 根据 proxyMode 创建对应 Agent
    │   └─ 将 httpsAgentRequestFields 作为 Agent 构造参数
    └─ 注入 httpAgent / httpsAgent 到请求配置
    ↓
5.  发送请求
    ↓
6.  重定向时重新执行步骤 4（用重定向后的 URL）
```

### 7.2 配置交互矩阵

| 场景 | CA 证书 | 客户端证书 | 代理配置 | 备注 |
|------|---------|-----------|---------|------|
| 普通 HTTPS 请求（无代理） | ✅ 用于验证服务器证书 | ✅ 用于服务器验证客户端 | ❌ | 直接使用 `https.Agent` |
| HTTPS → HTTP 代理 → HTTPS 目标 | ✅ 用于验证目标服务器证书 | ✅ 用于目标服务器验证客户端 | ✅ 代理凭据用于代理认证 | 通过 `PatchedHttpsProxyAgent` 转发 TLS 选项到目标连接 |
| HTTPS → HTTPS 代理 → HTTP 目标 | ✅ 用于验证代理服务器证书 | ❌ 目标是 HTTP，无需客户端证书 | ✅ 代理凭据 + TLS 验证代理 | `tlsOptions` 应用于代理连接 |
| HTTPS → HTTPS 代理 → HTTPS 目标 | ✅ 同时验证代理和目标服务器 | ✅ 用于目标服务器验证客户端 | ✅ 代理凭据 + TLS 验证代理 | CA 证书同时用于代理和目标连接 |
| 禁用 TLS 验证 | ❌ 不加载 CA 证书 | ✅ 仍可使用客户端证书（不推荐） | ✅ 代理不受影响 | `rejectUnauthorized = false` |
| SOCKS 代理 → HTTPS 目标 | ✅ 用于验证目标服务器 | ✅ 用于目标服务器验证客户端 | ✅ SOCKS 认证 | SOCKS 代理本身不涉及 TLS |

### 7.3 bru.sendRequest() 的配置继承

脚本中调用 `bru.sendRequest()` 时，会继承主请求的证书和代理配置：

```typescript
// send-request.ts:22-45
const createSendRequest = (config?: SendRequestConfig) => {
  return async (requestConfig) => {
    if (config) {
      const { httpAgent, httpsAgent } = await getHttpHttpsAgents({
        ...config,
        requestUrl: normalizedConfig.url  // 使用脚本请求的 URL
      });
      normalizedConfig.httpAgent = httpAgent;
      normalizedConfig.httpsAgent = httpsAgent;
    }
    // ...
  };
};
```

**关键点**：
- 证书配置完全继承（CA、客户端证书、TLS 验证设置）
- 代理配置继承，但代理绕过规则会**重新评估**脚本请求的 URL
- 客户端证书的域名匹配也会**重新匹配**脚本请求的 URL

### 7.4 OAuth2 令牌请求的配置继承

OAuth2 流程中获取令牌时，会为令牌 URL 和刷新 URL 单独评估配置：

```javascript
// index.js:179-226
if (accessTokenUrl && grantType !== 'implicit') {
  const tokenRequestForConfig = { ...requestCopy, url: interpolatedTokenUrl };
  certsAndProxyConfigForTokenUrl = await getCertsAndProxyConfig({
    ...,
    request: tokenRequestForConfig  // 使用令牌 URL
  });
}
```

这意味着：
- 如果令牌服务器域名匹配了某个客户端证书，会自动应用
- 代理绕过规则对令牌 URL 单独评估
- CA 证书配置保持一致

---

## 八、重定向时的配置重新评估

当请求发生重定向时，会对新 URL 重新评估代理和证书配置：

```javascript
// axios-instance.js:419-428
try {
  await setupProxyAgents({
    requestConfig,  // 包含重定向后的 URL
    proxyMode,
    proxyConfig,
    httpsAgentRequestFields,
    interpolationOptions,
    timeline
  });
} catch (err) {
  // 错误处理...
}
```

**重新评估的内容**：
1. **代理绕过**：新 URL 是否匹配 `no_proxy` 或 `bypassProxy` 规则
2. **客户端证书**：新 URL 的域名是否匹配其他客户端证书配置
3. **PAC 解析**：如果使用 PAC，会用新 URL 重新执行 PAC 脚本

**保持不变的内容**：
- CA 证书配置
- TLS 验证设置（`rejectUnauthorized`）
- 代理配置本身（`proxyMode`、`proxyConfig`）

---

## 九、关键代码文件索引

| 功能 | 文件路径 | 关键行 |
|------|---------|-------|
| CA 证书加载与合并 | `packages/bruno-requests/src/utils/ca-cert.ts` | 92-160 |
| 证书与代理配置获取（Electron） | `packages/bruno-electron/src/ipc/network/cert-utils.js` | 13-165 |
| Agent 创建（通用） | `packages/bruno-requests/src/utils/http-https-agents.ts` | 530-571 |
| Agent 创建（CLI） | `packages/bruno-cli/src/utils/proxy-util.js` | 105-198 |
| 代理配置格式转换 | `packages/bruno-requests/src/utils/proxy-util.ts` | 40-92 |
| Axios 实例与拦截器（Electron） | `packages/bruno-electron/src/ipc/network/axios-instance.js` | 74-502 |
| 脚本请求代理继承 | `packages/bruno-requests/src/scripting/send-request.ts` | 22-76 |
| 请求执行入口（Electron） | `packages/bruno-electron/src/ipc/network/index.js` | 101-359 |
| PAC 代理解析 | `packages/bruno-requests/src/utils/pac-resolver.ts` | - |
| Agent 缓存 | `packages/bruno-requests/src/utils/agent-cache.ts` | - |

---

## 十、总结

### 10.1 核心设计原则

1. **集中配置，分散执行**：`getCertsAndProxyConfig()` 统一聚合配置，实际 Agent 创建在请求拦截器中执行
2. **层级覆盖**：集合级配置 > 应用级配置 > 系统级配置
3. **URL 感知**：客户端证书匹配、代理绕过、PAC 解析都针对具体 URL 评估
4. **重定向安全**：重定向时重新评估 URL 相关配置，避免配置错配
5. **变量插值**：所有配置字段支持环境变量插值，增强灵活性

### 10.2 优先级总览

```
代理优先级：
  请求级 noproxy > 集合级代理 > 应用级代理 > 系统级代理

证书优先级（CA 合并顺序，后者覆盖前者）：
  系统 CA → Node 根 CA → 自定义 CA → NODE_EXTRA_CA_CERTS

客户端证书优先级：
  按配置顺序，第一个域名匹配者生效

TLS 验证开关优先级：
  shouldVerifyTls = false 时，忽略所有 CA 配置，rejectUnauthorized = false
```

### 10.3 常见组合场景

1. **企业内网 + 自定义 CA**：启用自定义 CA，`shouldKeepDefaultCerts=true`，同时信任内网和公网证书
2. **严格安全环境**：启用自定义 CA，`shouldKeepDefaultCerts=false`，仅信任企业内部 CA
3. **MTLS 双向认证**：配置客户端证书，按域名自动匹配适用的证书
4. **开发环境调试验证**：关闭 `shouldVerifyTls`，跳过证书验证（仅用于开发）
5. **PAC 自动代理**：配置 PAC 源，由脚本动态选择代理服务器
