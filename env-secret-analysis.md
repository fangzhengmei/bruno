# Bruno 环境变量与 Secret 解析机制分析报告

## 1. 概述

本报告详细分析 Bruno API 客户端中环境变量（Environment Variables）与敏感信息（Secrets）的读取、存储、注入及插值（Interpolation）机制。

---

## 2. 变量源分类与优先级

### 2.1 变量来源（共9类）

| 变量类型 | 来源位置 | 说明 |
|---------|---------|------|
| **globalEnvironmentVariables** | 全局环境配置 | 跨集合共享的环境变量 |
| **collectionVariables** | Collection 根级别 | 集合级别的请求前置变量（vars.req） |
| **envVariables** | Environment 环境配置 | 特定环境（如Dev/Prod）的变量 |
| **folderVariables** | Folder 文件夹级别 | 文件夹级别的请求前置变量 |
| **requestVariables** | Request 请求级别 | 单个请求的前置变量 |
| **oauth2CredentialVariables** | OAuth2 凭证 | 从 OAuth2 认证流程获取的动态凭证 |
| **runtimeVariables** | 运行时脚本设置 | 通过 `bru.setVar()` 等脚本API动态设置 |
| **promptVariables** | 交互式提示变量 | 请求发送前用户输入的变量 |
| **processEnvVars** | OS/文件系统 | 系统环境变量 + `.env` 文件变量 |

### 2.2 变量优先级（高 → 低）

```
promptVariables > runtimeVariables > oauth2CredentialVariables > requestVariables > folderVariables > envVariables > collectionVariables > globalEnvironmentVariables
```

**关键代码位置**：`packages/bruno-electron/src/ipc/network/interpolate-vars.js:47-61`

---

## 3. process.env 变量解析机制

### 3.1 三级来源与优先级

`process.env` 变量从三个位置读取，优先级（高 → 低）：

| 来源 | 优先级 | 说明 |
|-----|-------|------|
| **Collection .env** | ★★★ 最高 | 集合根目录下的 `.env` 文件 |
| **Workspace .env** | ★★ 中等 | 工作区根目录下的 `.env` 文件 |
| **OS process.env** | ★ 最低 | 操作系统的环境变量 |

### 3.2 核心实现代码

**文件位置**：`packages/bruno-electron/src/store/process-env.js:19-29`

```javascript
const getProcessEnvVars = (collectionUid) => {
  const workspacePath = collectionWorkspaceMap[collectionUid];
  const workspaceEnvVars = workspacePath ? workspaceDotEnvVars[workspacePath] : {};

  return {
    ...process.env,           // 最低优先级：OS环境变量
    ...workspaceEnvVars,      // 中等优先级：工作区 .env
    ...dotEnvVars[collectionUid]  // 最高优先级：集合 .env
  };
};
```

### 3.3 工作机制

1. **内存存储**：所有 `.env` 文件变量加载后存储在内存中
2. **集合隔离**：每个集合（collectionUid）的 `.env` 变量独立存储
3. **工作区关联**：通过 `collectionWorkspaceMap` 维护集合与工作区的映射关系

---

## 4. Secrets 安全存储机制

### 4.1 存储方案

Secrets（敏感变量如API密钥、密码等）不存储在明文配置文件中，而是：

1. **加密存储**：使用 `electron-store` 进行本地加密存储
2. **独立文件**：存储在系统用户目录下的 `secrets.json` 中
3. **集合隔离**：按集合路径 + 环境名称分组存储

### 4.2 数据结构

**文件位置**：`packages/bruno-electron/src/store/env-secrets.js:24-82`

```javascript
{
  "collections": [
    {
      "path": "/Users/xxx/project-collection",
      "environments": [
        {
          "name": "Local",
          "secrets": [
            {
              "name": "api_token",
              "value": "encrypted_value_here"  // 加密后的值
            }
          ]
        }
      ]
    }
  ]
}
```

### 4.3 核心方法

**文件位置**：`packages/bruno-electron/src/store/env-secrets.js`

| 方法 | 功能 | 位置 |
|-----|------|------|
| `storeEnvSecrets()` | 存储环境的所有 secret 变量（加密） | 第32-82行 |
| `getEnvSecrets()` | 获取指定环境的所有 secret 变量 | 第84-98行 |
| `renameEnvironment()` | 重命名环境时更新 secret 存储 | 第100-115行 |
| `deleteEnvironment()` | 删除环境时清除对应的 secret 数据 | 第117-127行 |

### 4.4 加密流程

1. 遍历环境的所有变量，筛选 `secret: true` 的变量
2. 使用 `encryptStringSafe()` 对值进行加密
3. 按集合 → 环境的层级结构存储到 electron-store

---

## 5. Secret 完整链路：从存储到请求组装

### 5.1 完整执行时序

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     Secret 完整注入链路                                    │
├──────────────────────────────────────────────────────────────────────────┤
│  1. 环境文件加载                                                          │
│     └─> parseEnvironmentFile() 读取 .yml 环境文件                         │
│         位置：workspace-environments.js:56-82 / collection-watcher.js     │
├──────────────────────────────────────────────────────────────────────────┤
│  2. Secret 读取                                                           │
│     └─> environmentSecretsStore.getEnvSecrets(collectionPath, environment)│
│         从 electron-store 读取该环境的所有加密 secrets                    │
│         位置：env-secrets.js:84-98                                        │
├──────────────────────────────────────────────────────────────────────────┤
│  3. Secret 解密                                                           │
│     └─> decryptStringSafe(secret.value)                                   │
│         使用系统安全存储 API 解密敏感值                                    │
│         位置：workspace-environments.js:75 / collection-watcher.js:123    │
├──────────────────────────────────────────────────────────────────────────┤
│  4. Secret 合并到环境变量                                                  │
│     └─> _.find(environment.variables, (v) => v.name === secret.name)      │
│         按变量名匹配，将解密后的值覆盖回环境变量对象                       │
│         位置：workspace-environments.js:72-78                             │
├──────────────────────────────────────────────────────────────────────────┤
│  5. 环境变量传递                                                          │
│     ├─> 通过 IPC 事件发送到渲染进程                                       │
│     └─> 或直接传递给 runRequest() 函数                                    │
├──────────────────────────────────────────────────────────────────────────┤
│  6. 环境变量格式转换                                                      │
│     └─> getEnvVars(environment)                                           │
│         将环境变量数组转换为 { name: value } 键值对格式                    │
│         位置：collection.js:749-768                                       │
├──────────────────────────────────────────────────────────────────────────┤
│  7. 变量插值（interpolateVars）                                           │
│     ├─> 7.1 克隆环境变量，防止修改原对象                                   │
│     ├─> 7.2 环境变量预解析（注意：此阶段 process.env 为空对象）            │
│     ├─> 7.3 合并所有变量源（含真实 processEnvVars）                        │
│     └─> 7.4 插值 URL/Headers/Body/Auth 等所有位置                         │
└──────────────────────────────────────────────────────────────────────────┘
```

### 5.2 关键环节代码证据

#### 环节 1-4：Secret 读取、解密、合并

**文件位置**：`packages/bruno-electron/src/store/workspace-environments.js:70-79`

```javascript
if (this.envHasSecrets(environment)) {
  const envSecrets = environmentSecretsStore.getEnvSecrets(workspacePath, environment);
  _.each(envSecrets, (secret) => {
    const variable = _.find(environment.variables, (v) => v.name === secret.name);
    if (variable && secret.value) {
      const decryptionResult = decryptStringSafe(secret.value);
      variable.value = decryptionResult.value;
    }
  });
}
```

**同样逻辑也存在于**：`packages/bruno-electron/src/app/collection-watcher.js:118-127`

#### 环节 6：环境变量格式转换

**文件位置**：`packages/bruno-electron/src/utils/collection.js:749-768`

```javascript
const getEnvVars = (environment = {}) => {
  const variables = environment.variables;
  if (!variables || !variables.length) {
    return {
      __name__: environment.name
    };
  }

  const envVars = {};
  each(variables, (variable) => {
    if (variable.enabled) {
      envVars[variable.name] = variable.value;
    }
  });

  return {
    ...envVars,
    __name__: environment.name
  };
};
```

---

## 6. 变量插值（Interpolation）核心机制

### 6.1 插值语法

使用双大括号语法：`{{variableName}}`

支持：
- 直接变量引用：`{{baseUrl}}/api/users`
- 嵌套对象访问：`{{user.profile.name}}`
- 嵌套插值：变量值中可包含其他变量引用
- Mock 数据函数：`{{$uuid}}`, `{{$timestamp}}`, `{{$randomInt}}` 等

### 6.2 插值引擎实现

**文件位置**：`packages/bruno-common/src/interpolate/index.ts`

#### 核心流程：

```
输入字符串 → Mock 数据预处理 → 循环匹配 {{...}} → 递归解析嵌套变量 → 输出最终字符串
```

#### 关键代码片段：

```typescript
const interpolate = (str: string, obj: Record<string, any>, options) => {
  // 1. 预处理：解析 mock 函数（如 {{$uuid}}）
  const preparedStr = prepareMock(str, escapeJSONStrings);
  
  // 2. 对变量对象也进行 mock 预处理
  const preparedObj = prepareMockObj(obj, escapeJSONStrings);
  
  // 3. 递归替换变量
  return replace(preparedStr, preparedObj);
};
```

### 6.3 ⚠️ 重要修正：环境变量预解析的真实情况

**之前的错误结论**：环境变量会先用 process.env 预解析

**真实代码实现**：`packages/bruno-electron/src/ipc/network/interpolate-vars.js:30-39`

```javascript
// envVars can inturn have values as {{process.env.VAR_NAME}}
// so we need to interpolate envVars first with processEnvVars
forOwn(envVariables, (value, key) => {
  envVariables[key] = interpolate(value, {
    process: {
      env: {
        // ⚠️ 这里是空对象！根本没有注入 processEnvVars！
        // 注释说"需要用 processEnvVars 先插值"，但实际传入空对象
      }
    }
  });
});
```

**真实结论**：
- 虽然代码注释说明"需要用 processEnvVars 先插值 envVars"
- 但**预解析阶段实际上传入的是空对象**，根本没有注入真实的 process.env 值！
- 真正的 process.env 注入只发生在后续的 `_interpolate` 函数中（第56-59行）
- 这是一个"代码注释与实际实现不一致"的情况

### 6.4 真正的 process.env 注入位置

**文件位置**：`packages/bruno-electron/src/ipc/network/interpolate-vars.js:56-60`

```javascript
const _interpolate = (str, { escapeJSONStrings } = {}) => {
  // ...
  const combinedVars = {
    // ... 其他变量
    process: {
      env: {
        ...processEnvVars  // ✅ 这里才真正注入了真实的 process.env 变量
      }
    }
  };
  return interpolate(str, combinedVars, { escapeJSONStrings });
};
```

### 6.5 JSON 字符串转义

当在 JSON Body 中插值时，自动对特殊字符转义：
- `\` → `\\`
- 换行符 → `\n`
- 制表符 → `\t`
- 双引号 → `\"`

**触发条件**：`escapeJSONStrings: true`

---

## 7. 请求发送前的变量注入流程

### 7.1 完整执行时序

```
┌─────────────────────────────────────────────────────────────┐
│                     请求准备阶段                               │
├─────────────────────────────────────────────────────────────┤
│  1. prepareRequest() - 构建请求对象                           │
│     └─> mergeVars() - 合并 collection/folder/request 级变量   │
├─────────────────────────────────────────────────────────────┤
│  2. runPreRequest() - 执行前置脚本                            │
│     ├─> ScriptRuntime - 运行 pre-request 脚本                 │
│     └─> 更新 runtimeVariables / envVariables                 │
├─────────────────────────────────────────────────────────────┤
│  3. interpolateVars() - 核心变量插值（关键步骤）              │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                     变量插值细节                              │
├─────────────────────────────────────────────────────────────┤
│  3.1 克隆环境变量，防止修改原对象                             │
│  3.2 环境变量预解析（⚠️ 注意：此阶段 process.env 为空对象）    │
│  3.3 合并所有变量源（✅ 此处注入真实 processEnvVars）         │
│  3.4 插值 URL                                               │
│  3.5 插值 Headers（键和值都支持变量）                        │
│  3.6 插值 Request Body                                       │
│      ├─> JSON: 整个 JSON 对象字符串化后整体插值               │
│      ├─> Form-urlencoded: 逐个字段插值                       │
│      ├─> Multipart: 逐个字段插值，支持文件字段                │
│      └─> Raw: 直接插值字符串                                 │
│  3.7 插值 Path Parameters                                    │
│  3.8 插值 Proxy 配置（协议、主机、端口、认证）                 │
│  3.9 插值 Auth 配置（Basic/Digest/OAuth1/OAuth2/AWS SigV4）  │
│  3.10 URL 编码（可选）                                       │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                     发送请求                                  │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 核心注入点代码分析

**文件位置**：`packages/bruno-electron/src/ipc/network/interpolate-vars.js`

#### 7.2.1 变量合并逻辑

```javascript
const combinedVars = {
  ...globalEnvironmentVariables,    // 优先级：最低
  ...collectionVariables,
  ...envVariables,
  ...folderVariables,
  ...requestVariables,
  ...oauth2CredentialVariables,
  ...runtimeVariables,
  ...promptVariables,               // 优先级：最高
  process: {
    env: {
      ...processEnvVars             // ✅ 此处注入真实的 process.env 变量
    }
  }
};
```

#### 7.2.2 各部分插值实现

| 插值对象 | 实现方式 | 特殊处理 | 位置 |
|---------|---------|---------|------|
| **URL** | 直接字符串插值 | 路径参数（:param）特殊处理 | 第68行 |
| **Headers** | 键和值分别插值 | Content-Type 影响 Body 插值方式 | 第71-74行 |
| **JSON Body** | JSON.stringify → 插值 → JSON.parse | 自动转义特殊字符 | 第116-131行 |
| **Form URL Encoded** | 遍历数组，逐个字段插值 | - | 第132-138行 |
| **Multipart** | 遍历数组，支持文件字段 | 不插值 Buffer | 第139-147行 |
| **Path Params** | 匹配 `:param` 格式替换 | 支持 OData 风格括号内参数 | 第153-213行 |
| **Proxy** | 协议、主机、端口、用户名、密码分别插值 | - | 第215-224行 |
| **Auth** | 所有认证字段插值 | Basic/Digest/OAuth1/OAuth2/AWS | 第228-381行 |

### 7.3 OAuth2 凭证变量注入

**特殊机制**：OAuth2 认证成功后，凭证会作为动态变量注入：

```javascript
// 变量命名格式
`$oauth2.${credentialsId}.${key}` = value

// 示例
`$oauth2.my-github-app.access_token` = "gho_xxxxxx"
```

**代码位置**：`packages/bruno-electron/src/ipc/network/index.js:896-898`

---

## 8. 变量作用域与生命周期

### 8.1 变量作用域层次

```
Global (跨集合)
    ↓
Collection (集合级)
    ↓
Environment (环境级)
    ↓
Folder (文件夹级)
    ↓
Request (请求级)
    ↓
Runtime (运行时/脚本级) → 单次请求有效
```

### 8.2 变量生命周期

| 变量类型 | 生命周期 | 持久化 |
|---------|---------|--------|
| globalEnvironmentVariables | 应用运行期间 | 是（集合配置文件） |
| collectionVariables | 集合打开期间 | 是（collection.bru） |
| envVariables | 环境激活期间 | 是（环境配置文件） |
| folderVariables | 请求执行期间 | 是（folder.bru） |
| requestVariables | 请求执行期间 | 是（请求文件） |
| oauth2CredentialVariables | 凭证有效期内 | 是（加密存储） |
| runtimeVariables | 单次请求/集合运行 | 否（内存） |
| promptVariables | 单次请求 | 否（内存） |
| processEnvVars | 应用运行期间 | 否（实时读取） |

---

## 9. 关键设计特性

### 9.1 嵌套插值支持

变量值可以包含其他变量引用，引擎会循环解析直到没有可替换的变量。

**示例**：
```
baseUrl = {{protocol}}://{{host}}:{{port}}
protocol = https
host = api.example.com
port = 443

→ 最终解析为：https://api.example.com:443
```

### 9.2 循环引用防护

使用 `visited` 集合记录已解析的变量，防止无限循环：

```typescript
if (patternRegex.test(replacement) && !visited.has(match)) {
  // 递归解析嵌套变量
  visited.add(match);
}
```

### 9.3 结果缓存优化

使用 `results` Map 缓存已解析的变量结果，提升重复插值的性能。

### 9.4 敏感信息安全

1. **Secret 独立存储**：不与普通环境变量一起存储在明文中
2. **加密落盘**：使用系统安全存储 API 加密存储
3. **内存保护**：插值完成后敏感信息仅在内存中短暂存在

---

## 10. 结论-证据对照

每条结论对应精确的代码位置，确保结论与实现一致：

| 结论 | 代码证据位置 | 状态 |
|-----|-------------|------|
| **process.env 三级优先级**：Collection .env > Workspace .env > OS process.env | `packages/bruno-electron/src/store/process-env.js:19-29` | ✅ 证实 |
| **Secret 加密存储**：使用 electron-store 独立存储，不落明文配置 | `packages/bruno-electron/src/store/env-secrets.js:24-82` | ✅ 证实 |
| **Secret 完整链路**：读取 → 解密 → 合并 → 转换 → 插值 | `packages/bruno-electron/src/store/workspace-environments.js:70-79` | ✅ 证实 |
| **⚠️ 重要修正**：环境变量预解析阶段 process.env 为空对象 | `packages/bruno-electron/src/ipc/network/interpolate-vars.js:30-39` | ✅ 证实（注释与实现不一致） |
| **真正的 process.env 注入位置**：在 `_interpolate` 函数的 combinedVars 中 | `packages/bruno-electron/src/ipc/network/interpolate-vars.js:56-60` | ✅ 证实 |
| **变量优先级顺序**：prompt > runtime > oauth2 > request > folder > env > collection > global | `packages/bruno-electron/src/ipc/network/interpolate-vars.js:47-61` | ✅ 证实 |
| **9 类变量源**：global/collection/env/folder/request/oauth2/runtime/prompt/process.env | `packages/bruno-electron/src/ipc/network/interpolate-vars.js:21-61` | ✅ 证实 |
| **嵌套插值支持**：循环解析直到无可替换变量 | `packages/bruno-common/src/interpolate/index.ts:89-130` | ✅ 证实 |
| **JSON 字符串自动转义**：对反斜杠、换行、制表、双引号转义 | `packages/bruno-common/src/interpolate/index.ts:21-32` | ✅ 证实 |
| **Secret 解密时机**：环境文件加载时即时解密合并 | `packages/bruno-electron/src/store/workspace-environments.js:75` | ✅ 证实 |

---

## 11. 相关文件索引

| 功能模块 | 文件路径 |
|---------|---------|
| 变量插值核心引擎 | `packages/bruno-common/src/interpolate/index.ts` |
| 请求变量注入 | `packages/bruno-electron/src/ipc/network/interpolate-vars.js` |
| Secret 安全存储 | `packages/bruno-electron/src/store/env-secrets.js` |
| Secret 解密合并 | `packages/bruno-electron/src/store/workspace-environments.js` |
| process.env 变量管理 | `packages/bruno-electron/src/store/process-env.js` |
| 环境变量格式转换 | `packages/bruno-electron/src/utils/collection.js:749-768` |
| 变量层级合并 | `packages/bruno-electron/src/utils/collection.js:mergeVars()` |
| 网络请求主流程 | `packages/bruno-electron/src/ipc/network/index.js` |
| 加密解密工具 | `packages/bruno-electron/src/utils/encryption.js` |
| 集合文件 watcher | `packages/bruno-electron/src/app/collection-watcher.js` |
| CLI 端变量插值 | `packages/bruno-cli/src/runner/interpolate-vars.js` |

---

## 12. 总结

Bruno 的环境变量与 Secret 解析机制设计特点：

1. **多层级变量源**：支持 9 种不同来源的变量，覆盖各种使用场景
2. **明确的优先级规则**：从全局到运行时，优先级清晰可预测
3. **安全的 Secret 管理**：加密存储、隔离保存，敏感信息不落明文
4. **强大的插值引擎**：支持嵌套插值、Mock 数据、JSON 安全转义
5. **全链路注入**：URL、Headers、Body、Auth、Proxy 等所有位置都支持变量
6. **脚本可编程**：支持通过脚本动态设置、修改变量，灵活度极高
7. **⚠️ 注意事项**：环境变量预解析阶段未注入真实 process.env（注释与实现不一致）
