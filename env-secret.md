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

### 2.2 请求体文件上传脱敏

**位置**：`packages/bruno-electron/src/utils/common.js`

```javascript
// parseDataFromRequest 函数
const parseDataFromRequest = (request) => {
  let requestDataString;

  // 文件上传请求体完全脱敏
  if (request.mode === 'file') {
    requestDataString = '<request body redacted>';  // 触发点：文件模式直接替换
  }
  // ... 其他模式正常处理
};
```

**触发条件**：当请求为文件上传模式时，请求体数据被完全脱敏为占位字符串，避免二进制文件内容进入日志或报告。

### 2.3 HTML 报告图片数据脱敏

**位置**：`packages/bruno-common/src/runner/reports/html/generate-report.ts`

```javascript
// 报告生成时对图片数据脱敏
return {
  ...result,
  request: {
    ...result.request,
    data: request?.data ? redactImageData(request?.data, requestContentType) : request?.data
  },
  response: {
    ...result.response,
    data: response?.data ? redactImageData(response?.data, responseContentType) : response?.data
  }
};
```

**触发条件**：检测到内容类型为图片时，对二进制数据进行脱敏处理，防止敏感图片数据泄露到 HTML 报告中。

### 2.4 导出脱敏

**位置**：`packages/bruno-app/src/utils/environments.js`

```javascript
// buildEnvVariable 函数
export const buildEnvVariable = ({ envVariable: obj, withUuid = false }) => {
  return {
    name: obj.name ?? '',
    value: !!obj.secret ? '' : (obj.value ?? ''),  // 敏感值清空
    type: 'text',
    enabled: obj.enabled !== false,
    secret: !!obj.secret
  };
};
```

**触发条件**：任何环境导出场景（JSON、Bruno 格式），标记为 `secret: true` 的变量 `value` 字段被置为空字符串。

### 2.5 敏感字段清单

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

### 2.6 脱敏边界总结

| 脱敏场景 | 触发时机 | 处理方式 | 边界说明 |
|---------|---------|---------|---------|
| UI 显示 | 用户查看密码字段时 | 掩码为 `*` | 内存中仍保留原值，仅视觉脱敏 |
| 文件上传请求 | 解析请求体时 | 替换为 `<request body redacted>` | 不影响实际网络发送，仅日志/存储用 |
| 图片数据 | 生成 HTML 报告时 | `redactImageData` 函数处理 | 仅图片类型响应被脱敏 |
| 环境导出 | 导出环境配置时 | 敏感变量 `value` 置空 | 内存中的运行时值不受影响 |
| 日志输出 | 控制台打印对象时 | 敏感字段自动截断为 `***` | 需结合具体日志实现 |

---

## 三、变量优先级合并机制

### 3.1 优先级链条（从低到高）

**位置**：`packages/bruno-electron/src/ipc/network/interpolate-vars.js`

```javascript
// 变量合并顺序：后面的覆盖前面的
const combinedVars = {
  ...globalEnvironmentVariables,    // 1. 全局环境变量 (最低优先级)
  ...collectionVariables,           // 2. 集合级变量
  ...envVariables,                  // 3. 选中环境变量
  ...folderVariables,               // 4. 文件夹级变量
  ...requestVariables,              // 5. 请求级变量
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

### 3.2 process.env 命名空间与同名变量覆盖规则

#### 规则 1：系统环境变量最高优先级

如果用户在 Bruno 环境中定义了变量 `FOO=bruno_value`，同时系统环境也有 `FOO=system_value`，则插值时 `{{FOO}}` 会解析为 `system_value`。

**原因**：`processEnvVars` 在对象展开顺序中位于最后，其同名属性会覆盖前面所有层级定义的同名变量。

#### 规则 2：显式访问与隐式合并

```javascript
// 两种访问系统环境变量的方式
1. {{FOO}}                 → 隐式：走完整合并链，process.env.FOO 可能覆盖其他层级
2. {{process.env.FOO}}     → 显式：直接访问系统环境，不经过变量合并链
```

**重要区别**：
- 隐式方式 `{{FOO}}` 可能被脚本运行时变量（如 `bru.setVar('FOO', 'runtime')`）再次覆盖
- 显式方式 `{{process.env.FOO}}` 始终读取系统环境原始值

#### 规则 3：同名变量层级覆盖示例

```javascript
// 假设各层级都定义了同名变量 DB_HOST：
globalEnvironmentVariables.DB_HOST = 'global-host'
collectionVariables.DB_HOST      = 'collection-host'   // 覆盖全局
envVariables.DB_HOST             = 'env-host'          // 覆盖集合
folderVariables.DB_HOST          = 'folder-host'       // 覆盖环境
requestVariables.DB_HOST         = 'request-host'      // 覆盖文件夹
runtimeVariables.DB_HOST         = 'runtime-host'      // 覆盖请求
processEnvVars.DB_HOST           = 'system-host'       // 最终生效值
```

**最终结果**：`{{DB_HOST}}` = `system-host`

### 3.3 变量插值流程

```
1. 预插值：环境变量本身先解析 {{process.env.VAR_NAME}} 语法
2. 合并变量：按优先级链条合并所有变量源
3. 请求插值：URL → Headers → Body → Auth 依次插值
4. 特殊处理：路径参数、代理配置、各类认证协议单独插值
```

### 3.4 运行时变量持久化的真实入口

**位置**：`packages/bruno-js/src/bru.js`

#### 核心区别：setVar vs setEnvVar

```javascript
// 1. setVar() - 仅操作运行时变量（非持久）
bru.setVar('temp_key', 'value');
// → 只写入 runtimeVariables 对象
// → 内存临时存储，请求间不共享，应用重启丢失

// 2. setEnvVar() - 操作环境变量（可选持久化）
bru.setEnvVar('api_key', 'value123', { persist: false });
// → 只写入 envVariables，不持久化

bru.setEnvVar('api_key', 'value456', { persist: true });
// → 同时写入 envVariables 和 persistentEnvVariables
// → persistentEnvVariables 通过 IPC 事件同步到 UI
// → 触发 'main:persistent-env-variables-update' 事件
```

#### 持久化工作流：

**位置**：`packages/bruno-electron/src/ipc/network/index.js`

```javascript
// 脚本执行完成后
if (requestScript?.length) {
  scriptResult = await scriptRuntime.runRequestScript(...);

  // 触发持久化变量同步
  mainWindow.webContents.send('main:persistent-env-variables-update', {
    persistentEnvVariables: scriptResult.persistentEnvVariables,
    collectionUid
  });
}
```

**持久化限制**：
- `persist: true` 时 `value` 必须是字符串类型，否则抛出异常
- 仅对当前选中环境生效，不影响其他环境
- 持久化变量仅在集合范围内可见

### 3.5 变量类型与生命周期总结

| 变量类型 | 设置方式 | 生命周期 | 持久化 | 作用域 |
|---------|---------|---------|-------|--------|
| 全局环境变量 | GUI 设置 | 跨会话持久 | 是 | 所有集合 |
| 集合变量 | 集合级 Vars | 跨会话持久 | 是 | 当前集合 |
| 环境变量 | 环境配置 | 跨会话持久 | 是 | 当前集合+环境 |
| 文件夹变量 | 文件夹 Vars | 跨会话持久 | 是 | 文件夹内请求 |
| 请求变量 | 请求 Vars | 跨会话持久 | 是 | 单个请求 |
| 运行时临时 | `bru.setVar()` | 单次请求 | 否 | 脚本执行期间 |
| 运行时持久 | `bru.setEnvVar(..., {persist:true})` | 跨会话 | 是 | 当前环境 |
| 提示变量 | 运行时输入 | 单次请求 | 否 | 单次运行 |
| 系统环境 | `process.env` | 系统级 | 系统管理 | 全局最高优先级 |

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
│  │  *** 输出   │    │  bru.setEnvVar│   │  └─ AES-256-CBC     │  │
│  └─────────────┘    └─────────────┘    └─────────────────────┘  │
│         │                    │                    │             │
│         ▼                    ▼                    ▼             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐  │
│  │ 文件上传脱敏│    │  变量插值器   │    │  敏感字段配置       │  │
│  │ <redacted>  │    │  process.env │    │  sensitiveFields    │  │
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

### 5.2 为什么系统环境变量优先级最高？

- **安全设计**：避免敏感密钥硬编码在集合文件中
- **CI/CD 友好**：流水线运行时可通过环境变量注入密钥
- **团队协作**：开发者本地环境变量不影响他人

### 5.3 为什么 setVar 和 setEnvVar 分开设计？

- **职责清晰**：临时计算值 vs 持久化环境配置
- **副作用可控**：`setVar` 无副作用，`setEnvVar` 可影响后续请求
- **安全边界**：持久化变量需用户显式声明，避免意外覆盖

### 5.4 为什么双重加密 fallback？

- **可用性优先**：系统级密钥链不可用时优雅降级
- **兼容性**：跨平台一致性（Windows/macOS/Linux 行为统一）
- **渐进增强**：支持安全存储的平台获得更高安全等级

---

## 六、安全最佳实践

1. **始终标记敏感变量**：将密码、密钥、Token 设为 `secret: true`
2. **优先使用系统环境注入**：`{{process.env.MY_SECRET}}` 比硬编码更安全
3. **不使用全局环境存储高敏信息**：优先使用集合级环境
4. **定期轮换密钥**：加密绑定机器 ID，机器变更需重新输入
5. **导出后检查**：确认导出文件中敏感字段 value 为空
6. **脚本持久化需谨慎**：`setEnvVar` 带 `persist:true` 会写入磁盘
7. **避免日志打印**：脚本中避免 `console.log` 敏感变量
8. **文件上传需注意**：二进制请求体在日志中自动脱敏，但实际发送不受影响

---

*文档版本：v2.0 | 更新日期：2024 | 修正点：运行时持久化真实入口、日志脱敏触发边界、process.env 覆盖规则*
