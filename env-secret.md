# Bruno 环境变量与机密管理机制

## 一、加密存储机制

### 1.1 存储架构

Bruno 采用**分离存储**策略，将明文配置与加密机密分开存储：

- **明文配置**：存储在 `.yml` 或 `.bru` 文件中，可版本控制
- **加密机密**：存储在系统级安全存储中，不进入版本控制

### 1.2 加密算法与密钥派生

#### 加密算法优先级：

```javascript
// encryption.js
1. Electron SafeStorage (系统级密钥链) → 优先使用
2. AES-256-CBC → 当系统级存储不可用时降级使用
```

#### 密钥派生策略：

```javascript
// 机器绑定密钥
machineIdSync() → 生成与硬件绑定的密钥
sha256(machineId) → 派生最终加密密钥

// 算法标识前缀
$00: → Electron SafeStorage 加密
$01: → AES-256-CBC 加密
```

### 1.3 两类环境的存储差异

#### A. 集合环境 (Collection Environments)

**位置**：`packages/bruno-electron/src/store/env-secrets.js`

- **明文文件**：`environments/{envName}.yml` → 仅存储非敏感变量
- **机密存储**：使用 `electron-store` 命名为 `secrets`，结构如下：

```javascript
{
  collections: [{
    path: "/path/to/collection",
    environments: [{
      name: "Production",
      secrets: [
        { name: "API_KEY", value: "$00:encrypted_hex_value" },
        { name: "DB_PASSWORD", value: "$00:encrypted_hex_value" }
      ]
    }]
  }]
}
```

**读写流程**：
1. 保存时：遍历变量，标记为 `secret` 的值加密后存入 secrets store
2. 读取时：从 yml 读取明文，从 secrets store 解密并合并机密值

#### B. 全局环境 (Global Environments)

**位置**：`packages/bruno-electron/src/store/global-environments.js`

- **存储位置**：`electron-store` 命名为 `global-environments`
- **存储方式**：所有变量加密后直接存储在单一 JSON 文件中

```javascript
// 写入前加密
variables.map(v => ({
  ...v,
  value: v.secret ? encrypt(v.value) : v.value
}))

// 读取后解密
variables.map(v => ({
  ...v,
  value: v.secret ? decrypt(v.value) : v.value
}))
```

### 1.4 安全边界

| 场景 | 安全措施 |
|------|----------|
| 跨机器迁移 | AES-256 密钥绑定机器 ID，迁移后机密需重新输入 |
| 文件导出 | 敏感变量值被清空，不随导出文件泄露 |
| 版本控制 | 机密不存入 git，仅明文配置可提交 |

---

## 二、脱敏处理范围

### 2.1 UI 层面脱敏：MaskedEditor

**位置**：`packages/bruno-app/src/utils/common/masked-editor.js`

#### 核心特性：

```javascript
// 两种掩码策略
1. 字符级掩码 (< 500 字符) → 逐字符替换为 '*'
2. 行级掩码 (>= 500 字符) → 整行批量掩码，优化性能

// 关键 API
enable()       → 启用掩码
disable()      → 禁用掩码，显示明文
destroy()      → 清理资源，防止内存泄漏
```

#### 多场景适配：

- 单行密码字段
- 多行密钥/证书内容
- JSON 格式机密配置
- XML 格式认证信息

### 2.2 导出脱敏

**位置**：`packages/bruno-app/src/utils/environments.js`

```javascript
// buildEnvVariable 函数
export const buildEnvVariable = ({ envVariable, withUuid = false }) => {
  return {
    name: obj.name ?? '',
    value: !!obj.secret ? '' : (obj.value ?? ''),  // 敏感值清空
    type: 'text',
    enabled: obj.enabled !== false,
    secret: !!obj.secret
  };
};
```

**效果**：导出的 JSON 文件中，`secret: true` 的变量 `value` 为空字符串

### 2.3 敏感字段清单

**位置**：`packages/bruno-app/src/components/Environments/EnvironmentSettings/EnvironmentList/EnvironmentDetails/EnvironmentVariables/constants.js`

```javascript
const sensitiveFields = [
  'request.auth.oauth2.clientSecret',      // OAuth2 客户端密钥
  'request.auth.basic.password',           // Basic 认证密码
  'request.auth.digest.password',          // Digest 认证密码
  'request.auth.wsse.password',            // WSSE 认证密码
  'request.auth.ntlm.password',            // NTLM 认证密码
  'request.auth.awsv4.secretAccessKey',    // AWS v4 签名密钥
  'request.auth.bearer.token'              // Bearer Token
];
```

### 2.4 日志脱敏

**位置**：贯穿整个应用

- 所有涉及密码、密钥、Token 的日志输出均做掩码处理
- 控制台输出敏感字段时自动截断为 `***`

---

## 三、变量优先级合并机制

### 3.1 优先级链条（从低到高）

**位置**：`packages/bruno-electron/src/ipc/network/interpolate-vars.js`

```javascript
// 变量合并顺序：后面的覆盖前面的
const combinedVars = {
  ...globalEnvironmentVariables,    // 1. 全局环境变量 (最低)
  ...collectionVariables,           // 2. 集合变量
  ...envVariables,                  // 3. 环境变量
  ...folderVariables,               // 4. 文件夹变量
  ...requestVariables,              // 5. 请求变量
  ...oauth2CredentialVariables,     // 6. OAuth2 凭证变量
  ...runtimeVariables,              // 7. 运行时变量 (脚本设置)
  ...promptVariables,               // 8. 提示变量 (用户输入)
  process: {
    env: {
      ...processEnvVars             // 9. 系统环境变量 (最高优先级)
    }
  }
};
```

### 3.2 优先级详解

| 层级 | 来源 | 作用域 | 生命周期 | 优先级 |
|------|------|--------|----------|--------|
| 全局环境 | 全局配置 | 所有集合 | 持久化 | 1 (最低) |
| 集合变量 | 集合级配置 | 当前集合 | 持久化 | 2 |
| 环境变量 | 当前选中环境 | 当前集合 | 持久化 | 3 |
| 文件夹变量 | 文件夹 Vars 标签 | 文件夹内请求 | 持久化 | 4 |
| 请求变量 | 请求 Vars 标签 | 单个请求 | 持久化 | 5 |
| OAuth2 凭证 | OAuth2 认证配置 | 单个请求 | 持久化/缓存 | 6 |
| 运行时变量 | Pre-request 脚本 | 单次请求 | 内存临时 | 7 |
| 提示变量 | 运行时用户输入 | 单次请求 | 内存临时 | 8 |
| 系统环境 | process.env | 全局 | 系统级 | 9 (最高) |

### 3.3 变量插值流程

```
1. 预插值：环境变量本身先解析 {{process.env.XXX}} 语法
2. 合并变量：按优先级链条合并所有变量源
3. 请求插值：URL → Headers → Body → Auth 依次插值
4. 特殊处理：路径参数、代理配置、各类认证协议单独插值
```

### 3.4 运行时变量持久化

**位置**：`packages/bruno-js/src/runtime/vars-runtime.js`

```javascript
// bru.setVar(name, value) 行为
1. 首先在 runtimeVariables 中设置 (内存)
2. 标记为 persistent 的变量同步写入 persistentEnvVariables
3. 持久化变量在请求间保持状态，非持久化仅单次有效
```

---

## 四、整体安全架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         Bruno 安全架构                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐  │
│  │   UI 层     │    │   逻辑层     │    │      存储层          │  │
│  │             │    │             │    │                     │  │
│  │ MaskedEditor│    │ interpolate │    │ electron-store      │  │
│  │ (*****)     │───▶│  vars 合并   │───▶│  ├─ secrets.json    │  │
│  │             │    │ 优先级链条   │    │  └─ global-envs.json│  │
│  │ 导出脱敏    │    │             │    │                     │  │
│  └─────────────┘    └─────────────┘    └─────────────────────┘  │
│         │                    │                    │             │
│         ▼                    ▼                    ▼             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐  │
│  │  日志脱敏    │    │  运行时变量   │    │  加密引擎           │  │
│  │  console.   │    │  bru.setVar  │    │  ├─ safeStorage     │  │
│  │  *** 输出   │    │  内存状态    │    │  └─ AES-256-CBC     │  │
│  └─────────────┘    └─────────────┘    └─────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 五、关键设计决策

### 5.1 为什么分离存储？

- **版本控制友好**：明文配置可安全提交 git
- **合规要求**：敏感凭据不进入代码仓库，满足安全合规
- **灵活迁移**：环境配置可跨机器共享，机密单独管理

### 5.2 为什么多层级变量？

- **关注点分离**：全局配置、集合配置、请求配置各司其职
- **覆盖灵活**：高层级可覆盖低层级默认值，适配多环境场景
- **动态能力**：运行时脚本可修改变量，实现复杂自动化场景

### 5.3 为什么双重加密 fallback？

- **可用性优先**：系统级密钥链不可用时优雅降级
- **兼容性**：跨平台一致性（Windows/macOS/Linux 行为统一）
- **渐进增强**：支持安全存储的平台获得更高安全等级

---

## 六、安全最佳实践

1. **始终标记敏感变量**：将密码、密钥、Token 设为 `secret: true`
2. **不使用全局环境存储高敏信息**：优先使用集合级环境
3. **定期轮换密钥**：加密绑定机器 ID，机器变更需重新输入
4. **导出后检查**：确认导出文件中敏感字段 value 为空
5. **避免日志打印**：脚本中避免 `console.log` 敏感变量

---

*文档版本：v1.0 | 更新日期：2024*
