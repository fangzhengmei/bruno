# Bruno Collection Runner 数据驱动流程深度分析 (R2)

---

## 1. 数据驱动迭代数据进入执行链机制

### 1.1 当前实现状态

从代码分析来看，Bruno 的数据驱动功能目前处于**类型预留阶段**，完整的数据驱动迭代执行功能尚未完全实现。

**类型定义预留** (`packages/bruno-common/src/runner/types/index.ts:101`):
```typescript
export type T_RunnerRequestExecutionResult = {
  iterationIndex?: number;      // 迭代索引
  iterationData?: any;          // todo - csv/json row data (当前行数据)
  // ... 其他字段
};
```

### 1.2 预期的数据驱动设计架构

基于现有 Runner 架构和类型预留，可以推断数据驱动的完整实现流程：

```
数据文件 (CSV/JSON)
     │
     ▼
┌──────────────────────────────────────────────────────────┐
│  数据解析层                                               │
│  - CSV 解析: 第一行为表头，后续行为数据行                  │
│  - JSON 解析: 数组格式，每个元素为一行数据                  │
└──────────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────┐
│  迭代控制器 (Runner 主循环增强)                          │
│  - 外层循环: 迭代数据行数                                │
│  - 内层循环: 按顺序执行请求集合                            │
│  - 每次迭代注入 iterationIndex 和 iterationData           │
└──────────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────┐
│  变量注入层                                               │
│  - 注入方式1: runtimeVariables['key'] = value            │
│  - 注入方式2: 特殊前缀变量 {{$data.key}}                  │
│  - 注入时机: Pre-request 脚本执行前 / 变量插值前           │
└──────────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────┐
│  执行链                                                   │
│  正常执行流程 (Pre-request → 请求发送 → Post-response ...) │
└──────────────────────────────────────────────────────────┘
```

### 1.3 数据驱动与执行链集成点

基于现有 Runner 代码结构，数据驱动需要在以下关键点集成：

**集成点 1: 主循环入口** (`packages/bruno-electron/src/ipc/network/index.js:1368`)
```javascript
// 当前单层循环
let currentRequestIndex = 0;
while (currentRequestIndex < folderRequests.length) {
  // ... 单请求执行
}

// 预期双层循环 (数据驱动)
const iterations = parseDataFile(dataFilePath);  // 解析数据文件
for (let iterationIndex = 0; iterationIndex < iterations.length; iterationIndex++) {
  const iterationData = iterations[iterationIndex];
  
  // 注入迭代数据到运行时变量
  runtimeVariables['$iterationIndex'] = iterationIndex;
  Object.assign(runtimeVariables, iterationData);  // 直接注入数据字段
  
  let currentRequestIndex = 0;
  while (currentRequestIndex < folderRequests.length) {
    // ... 正常执行逻辑，此时 {{变量名}} 可引用数据字段
  }
}
```

**集成点 2: Bru 实例扩展** (`packages/bruno-js/src/bru.js`)
```javascript
// 预期新增 API
class Bru {
  // 获取当前迭代索引
  getIterationIndex() { return this.runtimeVariables['$iterationIndex']; }
  
  // 获取当前迭代数据
  getIterationData() { return this.runtimeVariables['$iterationData']; }
  
  // 获取迭代数据字段
  getData(key) { return this.interpolate(this.runtimeVariables[key]); }
}
```

---

## 2. 跨请求变量传递机制

### 2.1 传递链路概览

```
请求 A 执行
     │
     ▼
Pre-request 脚本
  bru.setVar('token', 'abc123')  ──┐
     │                              │
     ▼                              │ 引用同一对象引用
变量插值替换                        │
  URL/Headers/Body 中的 {{token}}   │
     │                              │
     ▼                              │
发送 HTTP 请求                       │
     │                              │
     ▼                              │
Post-response 脚本                  │
  bru.setVar('userId', res.data.id) ─┤
     │                              │
     ▼                              │
Assertions / Tests                  │
  bru.setVar(...)                   │
     │                              │
     ▼                              │
请求 B 执行                         │
     │                              │
     ▼                              │
Pre-request 脚本 ◄──────────────────┘
  bru.getVar('token')      // 'abc123'
  bru.getVar('userId')     // 12345
```

### 2.2 核心实现原理

**关键: 对象引用传递**

在 Runner 主循环中，`runtimeVariables` 是**同一个对象引用**传递给每个请求的 Bru 实例：

```javascript
// packages/bruno-electron/src/ipc/network/index.js
// 主循环外部初始化一次
let runtimeVariables = {};  // ✅ 单个对象引用

let currentRequestIndex = 0;
while (currentRequestIndex < folderRequests.length) {
  // 每个请求创建新的 Bru 实例，但共享同一个 runtimeVariables 对象
  const bru = new Bru({
    runtimeVariables: runtimeVariables,  // ✅ 引用传递，不是值拷贝
    // ... 其他变量
  });
  
  // 请求 A: bru.setVar('x', 1) → runtimeVariables.x = 1
  // 请求 B: bru.getVar('x') → 从同一个 runtimeVariables 读取 → 1
}
```

**Bru 类中的实现** (`packages/bruno-js/src/bru.js:283-307`):
```javascript
setVar(key, value) {
  // ✅ 直接修改共享的 runtimeVariables 对象
  this.runtimeVariables[key] = value;  
}

getVar(key) {
  // ✅ 从同一个共享对象读取，并支持插值
  return this.interpolate(this.runtimeVariables[key]);
}
```

### 2.3 变量作用域与生命周期

| 变量类型 | 存储位置 | 生命周期 | 跨请求传递 |
|---------|---------|---------|-----------|
| **Runtime Variables** | `runtimeVariables` | Runner 执行全程 | ✅ 支持 |
| **Environment Variables** | `envVariables` | Runner 执行全程 | ✅ 支持 (需 `{persist: true}` 才持久化到文件) |
| **Collection Variables** | `collectionVariables` | 单次请求 | ❌ 不支持 (只读) |
| **Folder Variables** | `folderVariables` | 单次请求 | ❌ 不支持 (只读) |
| **Request Variables** | `requestVariables` | 单次请求 | ❌ 不支持 (只读) |
| **Global Env Variables** | `globalEnvironmentVariables` | Runner 执行全程 | ✅ 支持 (部分实现) |

**重要说明**:
- `bru.setEnvVar(key, value, {persist: true})` 会同时修改内存中的变量和标记为持久化
- 持久化变量在 Runner 执行结束后会回写到环境文件
- Collection/Folder/Request Variables 当前仅支持读取，不支持脚本写入 (UI 同步问题待解决)

---

## 3. Pre-request 场景下 stopExecution 生效时序与中断边界

### 3.1 时序图 (精确到代码行)

```
packages/bruno-electron/src/ipc/network/index.js

行 1458: try {
行 1459:   preRequestScriptResult = await runPreRequest(...)
行 1460-1470:     │
行 1471: } catch (error) { preRequestError = error }
           │
           ▼  [脚本内部执行 bru.stopExecution()]
行 1511: if (preRequestScriptResult?.stopExecution) {
行 1512:   stopRunnerExecution = true;  // 设置标志位
行 1513: }
           │  ⚠️  注意: 此处不立即中断，仅设置标志
           ▼
行 1515: if (preRequestScriptResult?.skipRequest) {
行 1516-1529:   // 跳过当前请求，发送 skipped 事件，continue 循环
           │
           ▼  ✅ 继续执行后续流程
行 1532: const { data: requestData, dataBuffer } = parseDataFromRequest(request)
行 1542: let requestSent = { ... }
行 1554: mainWindow.webContents.send('main:run-folder-event', { type: 'request-sent' })
行 1598: timeStart = Date.now()
行 1599: let response = await axiosInstance(request)  // ✅ HTTP 请求仍会发送!
行 16xx: // ... 处理响应
行 17xx: // ... 执行 Post-response 脚本
行 17xx: // ... 执行 Assertions
行 18xx: // ... 执行 Tests
           │
           ▼  ✅ 直到所有测试执行完毕后才检查
行 1863: if (stopRunnerExecution) {
行 1864:   deleteCancelToken(cancelTokenUid)
行 1865:   mainWindow.webContents.send('main:run-folder-event', {
行 1866:     type: 'testrun-ended',
行 1867:     statusText: 'collection run was terminated!'
行 1868:   })
行 1872:   break;  // ✅ 真正中断: 跳出主 while 循环
行 1873: }
```

### 3.2 关键发现: 中断不是立即的

**⚠️ 重要结论**: `bru.stopExecution()` 在 Pre-request 中调用时：

1. **不会**立即中断当前请求的执行
2. 只会设置一个标志位 `stopRunnerExecution = true`
3. 当前请求会完整执行到底（包括：发送 HTTP 请求、Post-response 脚本、Assertions、Tests）
4. **真正的中断发生在当前请求全部执行完毕后**，才会跳出 while 循环，不再执行下一个请求

### 3.3 中断边界精确界定

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          单个请求执行边界                                 │
├─────────────────────────────────────────────────────────────────────────┤
│  Pre-request 脚本阶段                                                    │
│    ┌──────────────────────────────────────────────────────────┐         │
│    │  bru.stopExecution() → 设置 stopRunnerExecution = true   │         │
│    │  bru.skipRequest() → 立即 continue (跳过当前请求)         │ ◄───┐   │
│    └──────────────────────────────────────────────────────────┘     │   │
│                        │                                             │   │
│                        ▼  ⚠️ stopExecution 不会在此处中断               │   │
│  HTTP 请求发送阶段                                                    │   │
│    ┌──────────────────────────────────────────────────────────┐     │   │
│    │  变量插值替换                                              │     │   │
│    │  发送实际 HTTP 请求 (axios)                                │     │   │
│    │  接收响应                                                 │     │   │
│    └──────────────────────────────────────────────────────────┘     │   │
│                        │                                             │   │
│                        ▼                                             │   │
│  Post-response 脚本阶段                                               │   │
│    ┌──────────────────────────────────────────────────────────┐     │   │
│    │  执行 Post-response 脚本                                  │     │   │
│    └──────────────────────────────────────────────────────────┘     │   │
│                        │                                             │   │
│                        ▼                                             │   │
│  Assertions 阶段                                                      │   │
│    ┌──────────────────────────────────────────────────────────┐     │   │
│    │  运行所有断言                                             │     │   │
│    └──────────────────────────────────────────────────────────┘     │   │
│                        │                                             │   │
│                        ▼                                             │   │
│  Tests 脚本阶段                                                       │   │
│    ┌──────────────────────────────────────────────────────────┐     │   │
│    │  运行测试脚本                                             │     │   │
│    └──────────────────────────────────────────────────────────┘     │   │
│                        │                                             │   │
│                        ▼                                             │   │
│  中断检查点                                                          │   │
│    ┌──────────────────────────────────────────────────────────┐     │   │
│    │  if (stopRunnerExecution) { break; }  ◄────────────────────────┘   │
│    │  → 删除 Cancel Token                                              │
│    │  → 发送 testrun-ended 事件                                       │
│    │  → 跳出主 while 循环                                              │
│    └──────────────────────────────────────────────────────────┘         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.4 stopExecution vs skipRequest 行为对比

| 行为 | 触发指令 | 中断时机 | 当前请求 | 后续请求 |
|------|---------|---------|---------|---------|
| **跳过当前** | `bru.skipRequest()` | 立即 | ❌ 不执行 | ✅ 继续执行 |
| **终止 Runner** | `bru.stopExecution()` | 当前请求结束后 | ✅ 完整执行 | ❌ 不再执行 |

### 3.5 代码验证

**Pre-request 中 stopExecution 标志设置** (`行 1511-1513`):
```javascript
if (preRequestScriptResult?.stopExecution) {
  stopRunnerExecution = true;  // 仅设置标志，不做任何其他操作
}

// 没有任何条件判断阻止后续代码执行！
// 👇 这些代码一定会执行
const { data: requestData, dataBuffer: requestDataBuffer } = parseDataFromRequest(request);
let requestSent = { ... };
mainWindow.webContents.send('main:run-folder-event', { type: 'request-sent', ... });

timeStart = Date.now();
response = await axiosInstance(request);  // HTTP 请求一定会发送
```

**真正的中断检查** (`行 1863-1873`):
```javascript
// 在所有请求处理完成后才检查
if (stopRunnerExecution) {
  deleteCancelToken(cancelTokenUid);
  mainWindow.webContents.send('main:run-folder-event', {
    type: 'testrun-ended',
    collectionUid,
    folderUid,
    statusText: 'collection run was terminated!',
    runCompletionTime: new Date().toISOString()
  });
  break;  // 真正中断: 跳出 while 循环
}
```

### 3.6 CLI 版本的行为一致性

**CLI Runner 也遵循同样的逻辑** (`packages/bruno-cli/src/runner/run-single-request.js`):

```javascript
// Pre-request 中设置标志
if (result?.stopExecution) {
  shouldStopRunnerExecution = true;  // 仅设置标志
}

// 当前请求完整执行后，在返回时携带标志
return {
  // ... 结果
  shouldStopRunnerExecution  // 由调用者决定是否终止
};

// Runner 主循环在收到结果后检查标志
const result = await runSingleRequest(...);
if (result.shouldStopRunnerExecution) {
  break;  // 终止执行
}
```

---

## 4. 设计意图与潜在问题

### 4.1 为什么 stopExecution 不是立即中断？

**设计意图推测**:
1. **结果完整性**: 确保当前请求的执行结果（响应数据、测试结果等）能够被完整记录
2. **清理工作**: 允许 Post-response 脚本执行必要的清理逻辑
3. **一致性**: Pre-request 和 Post-response 中调用 `stopExecution` 行为一致
4. **避免半完成状态**: 中断正在进行的 HTTP 请求可能导致资源泄漏或状态不一致

### 4.2 潜在问题

1. **用户预期不符**: 用户可能期望调用 `stopExecution()` 后立即停止，包括不发送 HTTP 请求
2. **无法中止请求发送**: 如果 Pre-request 检测到严重错误（如认证失效），仍然会发送无效请求
3. **缺少立即中止 API**: 当前没有提供 `bru.abortRequest()` 类的立即中止 API

### 4.3 改进建议

```javascript
// 建议新增 API
class Bru {
  // 立即中止当前请求并终止 Runner
  abortRunner(message = 'Aborted by script') {
    this.abortRequest = true;
    this.stopExecution = true;
    this.abortMessage = message;
  }
  
  // 只中止当前请求，继续执行下一个
  abortCurrentRequest(message = 'Request aborted') {
    this.abortRequest = true;
    this.abortMessage = message;
  }
}
```

---

## 5. 总结

### 5.1 数据驱动迭代
- **当前状态**: 类型已预留，功能待实现
- **预期架构**: 外层数据行循环 + 内层请求执行循环
- **变量注入**: 通过 `runtimeVariables` 共享对象注入迭代数据

### 5.2 跨请求变量传递
- **核心机制**: 单个 `runtimeVariables` 对象引用在所有请求间共享
- **实现方式**: Bru 实例通过引用访问同一个变量容器
- **作用域**: Runtime/Environment 变量全程共享，Collection/Folder/Request 变量只读

### 5.3 stopExecution 生效时序
- **⚠️ 关键结论**: Pre-request 中调用不会立即中止，当前请求会完整执行
- **中断点**: 在当前请求的所有阶段（HTTP 发送、Post-response、Assertions、Tests）完成后
- **边界**: 跳出主 while 循环，不再处理下一个请求
- **对比**: `skipRequest()` 是立即跳过当前请求，`stopExecution()` 是完成当前后终止

---

**报告版本**: R2  
**分析日期**: 2026-05-17  
**代码范围**: packages/bruno-electron, packages/bruno-js, packages/bruno-common, packages/bruno-cli
