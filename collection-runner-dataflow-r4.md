# Bruno Collection Runner 数据驱动流程深度分析 (R4)

---

## 1. 核心结论（基于源码证据）

### 1.1 数据驱动能力真相

✅ **类型定义已预留**  
❌ **执行逻辑完全未落地**  
❌ **迭代级字段未在结果构建阶段被透传**

> **一句话总结**: 目前 Bruno 的 Collection Runner 只有**单层请求循环**，没有外层数据迭代循环。`T_RunnerResults`、`iterationIndex`、`iterationData` 等类型定义仅为**面向未来的接口预留**，在实际执行链路中从未被真正使用。

---

## 2. 迭代级字段透传真相：完整证据链

### 2.1 证据 1：CLI 主循环 - 只有单层执行

**文件**: `packages/bruno-cli/src/commands/run.js`

```javascript
// 行 680-765: 只有单层 while 循环，直接遍历请求
let currentRequestIndex = 0;
while (currentRequestIndex < requestItems.length) {
  const requestItem = cloneDeep(requestItems[currentRequestIndex]);
  
  // 执行单个请求
  const result = await runSingleRequest(requestItem, ...);
  
  // 直接收集结果 - 没有 iterationIndex 或 iterationData
  results.push({
    ...result,
    runDuration: process.hrtime(start)[0] + process.hrtime(start)[1] / 1e9,
    suitename: pathname.replace('.bru', ''),
    name,
    path: result.test?.filename || path.relative(collectionPath, pathname)
  });
  
  // 检查 bail、stopExecution、nextRequestName
  // ...
  
  currentRequestIndex++;
}

// 行 770: 直接调用 getRunnerSummary(results) - 单次汇总
const summary = printRunSummary(results);
```

**关键发现**:
- `results` 数组是**扁平的** `T_RunnerRequestExecutionResult[]`
- 没有外层循环遍历数据行
- `iterationIndex`、`iterationData` 字段**完全没有被赋值**

### 2.2 证据 2：Electron 主进程 - 同样只有单层执行

**文件**: `packages/bruno-electron/src/ipc/network/index.js`

```javascript
// 全局搜索: iterationIndex、iterationData
// 结果: 0 处匹配！
```

**结论**: GUI 版本的 Collection Runner 同样没有任何迭代逻辑。

### 2.3 证据 3：HTML Reporter 适配层 - 伪造迭代结构

**文件**: `packages/bruno-cli/src/reporters/html.js`

```javascript
// 行 5-18: 为了兼容 HTML 报告模板，手动包装成单层迭代
const makeHtmlOutput = async (results, outputPath, runCompletionTime, environment = null) => {
  let runnerResults = results;
  if (!results) {
    runnerResults = [];
  } else if (results.results) {
    // 🔴 适配代码: 将 CLI 单层结果包装成 T_RunnerResults 格式
    runnerResults = [{
      iterationIndex: 0,        // 硬编码为 0
      results: results.results, // 直接透传请求结果
      summary: results.summary  // 直接透传汇总
      // ❌ iterationData 完全缺失！
    }];
  } else if (Array.isArray(results)) {
    runnerResults = results;
  }

  const htmlString = generateHtmlReport({
    runnerResults: runnerResults,  // 传入适配后的假迭代结构
    // ...
  });
};
```

**关键发现**:
- `iterationIndex` 被**硬编码为 0**
- `iterationData` **完全缺失**，从未被设置
- 这是一个**兼容层 hack**，不是真正的数据驱动实现

### 2.4 证据 4：类型定义与实际使用的鸿沟

**文件**: `packages/bruno-common/src/runner/types/index.ts`

```typescript
// 🔵 类型定义层（面向未来）
export type T_RunnerResults = {
  iterationIndex: number;       // ✓ 已定义
  iterationData?: any;          // ✓ 已定义 (todo 注释)
  results: T_RunnerRequestExecutionResult[];  // ✓ 已定义
  summary: T_RunSummary;        // ✓ 已定义
};

export type T_RunnerRequestExecutionResult = {
  iterationIndex: number;       // ✓ 已定义
  // ... 其他字段
  // ❌ iterationData 不在此处
};

// 🔴 实际运行时
// - GUI: 单层请求循环，无迭代概念
// - CLI: 单层请求循环，无迭代概念
// - 结果: T_RunnerRequestExecutionResult[]（扁平数组）
```

### 2.5 证据 5：汇总函数签名 - 只处理请求级

**文件**: `packages/bruno-common/src/runner/runner-summary.ts`

```typescript
// 函数签名: 只接收请求级结果数组
export const getRunnerSummary = (results: T_RunnerRequestExecutionResult[]): T_RunSummary
```

**结论**: 汇总函数工作在请求级，不是迭代级。

---

## 3. 当前真实的汇总链路（无数据驱动）

### 3.1 完整执行流程图（当前实际状态）

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Bruno Collection Runner 真实执行链路                │
└─────────────────────────────────────────────────────────────────────────┘

阶段 1: 初始化
  ├── 解析命令行参数 / GUI 配置
  ├── 加载环境变量、全局变量
  ├── 收集并过滤请求（递归、Tag 过滤、tests-only 过滤）
  ├── 得到扁平的请求队列: requestItems = [Req1, Req2, Req3, ...]

阶段 2: 单层 while 循环（没有外层迭代！）
  let currentRequestIndex = 0;
  let results = [];
  
  while (currentRequestIndex < requestItems.length) {
    ┌─────────────────────────────────────────────────────┐
    │  单个请求执行流水线                                   │
    ├─────────────────────────────────────────────────────┤
    │  1. Pre-request 脚本                                 │
    │     - 检查 skipRequest / stopExecution / nextRequest │
    │  2. 变量插值替换                                      │
    │  3. 发送 HTTP 请求                                    │
    │  4. Post-response 脚本                                │
    │  5. Assertions 断言                                   │
    │  6. Tests 测试脚本                                    │
    └─────────────────────────────────────────────────────┘
    
    // 🔴 收集结果 - 没有迭代上下文！
    results.push({
      ...result,
      runDuration,
      name,
      path
      // ❌ 没有 iterationIndex
      // ❌ 没有 iterationData
    });
    
    // 检查控制流
    if (bail && failure) break;
    if (result.shouldStopRunnerExecution) break;
    if (result.nextRequestName) { /* 跳转逻辑 */ }
    else currentRequestIndex++;
  }

阶段 3: 汇总（单次汇总，不是按迭代汇总）
  const summary = getRunnerSummary(results);
  // summary 包含 totalRequests、passedRequests、failedRequests 等

阶段 4: 输出
  ├── 控制台打印 summary
  ├── JSON 报告: { summary, results }
  ├── JUnit 报告: 基于 results 生成
  └── HTML 报告: 经过适配层包装成假的 T_RunnerResults[]
       └── [{ iterationIndex: 0, results, summary }]  // 硬编码！
```

### 3.2 实际数据结构 vs 预留类型

```
实际运行时结构 (CLI/GUI 输出):
{
  summary: T_RunSummary,              // 全量汇总
  results: [                          // 扁平数组，无迭代分组
    { name: "Req1", status: "pass", ... },  // ❌ 无 iterationIndex
    { name: "Req2", status: "fail", ... },  // ❌ 无 iterationIndex
    ...
  ]
}

─────────────────────────────────────────────────

预留类型结构 (T_RunnerResults[] - 从未真正生成):
[
  {
    iterationIndex: 0,
    iterationData: { userId: 1001 },
    summary: T_RunSummary,            // 迭代 0 的汇总
    results: [
      { name: "Req1", iterationIndex: 0, ... },
      { name: "Req2", iterationIndex: 0, ... }
    ]
  },
  {
    iterationIndex: 1,
    iterationData: { userId: 1002 },
    summary: T_RunSummary,            // 迭代 1 的汇总
    results: [
      { name: "Req1", iterationIndex: 1, ... },
      { name: "Req2", iterationIndex: 1, ... }
    ]
  }
]
```

---

## 4. 类型与数据结构映射校正（最终版）

### 4.1 字段归属表（已核对源码）

| 字段 | 类型定义位置 | 实际使用位置 | 当前状态 |
|------|------------|------------|---------|
| `T_RunnerRequestExecutionResult.iterationIndex` | 类型定义已声明 | 执行代码中**从未赋值** | ⚠️ 预留 |
| `T_RunnerResults.iterationIndex` | 类型定义已声明 | 仅在 HTML 适配层**硬编码为 0** | ⚠️ 预留 |
| `T_RunnerResults.iterationData` | 类型定义已声明（含 todo 注释） | **从未被使用** | ⚠️ 预留 |
| `T_RunnerResults.results` | 类型定义已声明 | HTML 适配层透传请求结果 | ⚠️ 预留 |
| `T_RunnerResults.summary` | 类型定义已声明 | HTML 适配层透传全量汇总 | ⚠️ 预留 |
| `T_RunnerRequestExecutionResult` 其他字段 | 类型定义已声明 | 正常使用 | ✅ 已实现 |
| `getRunnerSummary()` | 类型定义已声明 | 正常调用 | ✅ 已实现 |

### 4.2 接口与实现的三层鸿沟

```
┌─────────────────────────────────────────────────────────────────────┐
│  层 1: 类型定义 (types/index.ts)                                      │
│  - T_RunnerResults (含 iterationIndex、iterationData)                 │
│  - 面向数据驱动的完整结构                                              │
└───────────────────────────────────┬───────────────────────────────────┘
                                    │ 鸿沟 1: 类型设计超前
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  层 2: 核心执行逻辑 (network/index.js, run.js)                        │
│  - 单层 while 循环                                                    │
│  - 没有数据文件解析、没有外层迭代                                      │
│  - 结果扁平数组: T_RunnerRequestExecutionResult[]                     │
└───────────────────────────────────┬───────────────────────────────────┘
                                    │ 鸿沟 2: 执行层无迭代
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  层 3: 输出适配层 (reporters/html.js)                                 │
│  - 硬编码 iterationIndex = 0                                         │
│  - 伪造 T_RunnerResults 格式                                          │
│  - iterationData 完全缺失                                             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5. 跨请求变量传递机制（保持不变，已核实验证）

### 5.1 源码证据

**文件**: `packages/bruno-cli/src/commands/run.js:383`

```javascript
// 主循环外部初始化一次
const runtimeVariables = {};  // ✅ 单个对象引用

// 行 681-699: 每个请求共享同一个 runtimeVariables
while (currentRequestIndex < requestItems.length) {
  const result = await runSingleRequest(
    requestItem,
    collectionPath,
    runtimeVariables,  // ✅ 引用传递给每个请求
    envVars,
    // ...
  );
}
```

**文件**: `packages/bruno-js/src/bru.js:283-307`

```javascript
setVar(key, value) {
  this.runtimeVariables[key] = value;  // ✅ 直接修改共享对象
}

getVar(key) {
  return this.interpolate(this.runtimeVariables[key]);  // ✅ 从共享对象读取
}
```

### 5.2 结论（已核实验证）

✅ **跨请求变量传递机制已完整实现**  
✅ **通过 runtimeVariables 对象引用共享实现**  
✅ **变量在整个 Runner 执行全程有效**

---

## 6. Pre-request stopExecution 生效时序（保持不变，已核实验证）

### 6.1 源码证据

**文件**: `packages/bruno-electron/src/ipc/network/index.js:1511-1513, 1863-1873`

```javascript
// 行 1511-1513: Pre-request 后只设置标志，不中断
if (preRequestScriptResult?.stopExecution) {
  stopRunnerExecution = true;  // 仅设置标志
}
// 👇 后续代码继续执行: 变量插值、发送请求、Post-response、Assertions、Tests...

// 行 1863-1873: 所有执行完毕后才检查
if (stopRunnerExecution) {
  deleteCancelToken(cancelTokenUid);
  mainWindow.webContents.send('main:run-folder-event', {
    type: 'testrun-ended',
    statusText: 'collection run was terminated!'
  });
  break;  // ✅ 真正中断点
}
```

### 6.2 结论（已核实验证）

✅ **stopExecution 不会立即中止当前请求**  
✅ **当前请求会完整执行到底**  
✅ **真正中断发生在当前请求全部执行完毕后**

---

## 7. 总结与启示

### 7.1 数据驱动能力现状总结

| 能力 | 状态 | 说明 |
|------|------|------|
| 类型定义 | ✅ 已完成 | `T_RunnerResults`、`iterationData` 等已定义 |
| 数据文件解析 | ❌ 未实现 | 没有 CSV/JSON 解析逻辑 |
| 外层迭代循环 | ❌ 未实现 | 只有单层请求循环 |
| 迭代数据注入 | ❌ 未实现 | 没有注入 iterationData 到 runtimeVariables |
| 迭代级汇总 | ❌ 未实现 | `getRunnerSummary()` 是全量汇总，不是按迭代 |
| 迭代结果结构 | ⚠️ 半实现 | HTML 报告硬编码适配，iterationData 缺失 |
| 跨请求变量传递 | ✅ 已实现 | runtimeVariables 引用共享 |
| stopExecution 时序 | ✅ 已实现 | 当前请求完成后终止 |

### 7.2 为什么会有这个设计？

**合理推测**:
1. Bruno 团队**规划了数据驱动功能**，先定义了类型接口
2. 优先实现了**核心的单轮 Runner 功能**（请求执行、变量传递、测试断言）
3. **数据驱动作为高阶功能**，计划后续迭代实现
4. 类型定义提前落地，可以**避免未来重构**，也为插件/扩展提供了预期接口

### 7.3 数据驱动落地需要的改造点

如果未来要实现完整的数据驱动功能，需要改造：

1. **CLI 参数扩展**: 新增 `--data-file`、`--iterations` 等参数
2. **数据解析层**: 新增 CSV/JSON 解析模块
3. **主循环改造**: 改为双层循环（外层迭代，内层请求）
4. **迭代上下文注入**: 每次迭代开始前注入 iterationData 到 runtimeVariables
5. **结果结构改造**: 按迭代分组收集结果，填充 iterationIndex、iterationData
6. **汇总逻辑改造**: 支持按迭代汇总 + 全量汇总
7. **报告适配**: 移除 HTML 报告的硬编码适配

---

## 8. 版本变更记录

| 版本 | 主要变更 | 日期 | 关键证据 |
|------|---------|------|---------|
| R1 | 完整流程分析 | 2026-05-17 | 主循环、执行流程 |
| R2 | 补充数据驱动、跨请求传递、stopExecution 时序 | 2026-05-17 | bru.js 源码 |
| R3 | 校正 iterationData 归属（请求层 → 迭代层） | 2026-05-17 | types/index.ts |
| **R4** | **✓ 核准汇总链路，确认数据驱动仅为类型预留**<br>**✓ 补充证据链：CLI 主循环、Electron 主进程、HTML 适配层**<br>**✓ 明确迭代级字段未在结果构建阶段被透传** | **2026-05-17** | **run.js、html.js、network/index.js** |

---

**报告版本**: R4  
**分析日期**: 2026-05-17  
**代码范围**: packages/bruno-common, packages/bruno-electron, packages/bruno-js, packages/bruno-cli  
**核心证据文件**:
- `packages/bruno-cli/src/commands/run.js` (CLI 主循环 - 单层执行)
- `packages/bruno-cli/src/reporters/html.js` (HTML 适配层 - 硬编码迭代)
- `packages/bruno-electron/src/ipc/network/index.js` (Electron 主进程 - 无迭代字段)
- `packages/bruno-common/src/runner/types/index.ts` (类型定义 - 面向未来)
- `packages/bruno-common/src/runner/runner-summary.ts` (汇总函数 - 请求级)
