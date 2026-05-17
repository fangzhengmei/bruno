# Bruno Collection Runner 数据驱动流程深度分析 (R3)

---

## 1. 类型与数据结构映射校正

### 1.1 R2 报告错误声明（已纠正）

❌ **之前的错误描述**:
> 迭代结果里 `iterationData` 挂在 `T_RunnerRequestExecutionResult` 上

✅ **正确的类型定义** (`packages/bruno-common/src/runner/types/index.ts`):

```typescript
// ┌─────────────────────────────────────────────────────────────────────────┐
// │  单个请求的执行结果 - T_RunnerRequestExecutionResult                      │
// └─────────────────────────────────────────────────────────────────────────┘
export type T_RunnerRequestExecutionResult = {
  iterationIndex: number;      // ✅ 存在
  name: string;
  path: string;
  request: T_EmptyRequest | T_Request;
  response: T_EmptyResponse | T_Response | T_SkippedResponse;
  status: null | undefined | string;
  error: null | undefined | string;
  assertionResults?: T_AssertionResult[];
  testResults?: T_TestResult[];
  preRequestTestResults?: T_TestResult[];
  postResponseTestResults?: T_TestResult[];
  runDuration: number;
  // ❌ iterationData 不在此处！
};

// ┌─────────────────────────────────────────────────────────────────────────┐
// │  单次迭代的全部结果 - T_RunnerResults                                      │
// └─────────────────────────────────────────────────────────────────────────┘
export type T_RunnerResults = {
  iterationIndex: number;       // ✅ 迭代索引
  iterationData?: any;          // ✅ CSV/JSON 行数据（仅迭代级别有）
  results: T_RunnerRequestExecutionResult[];  // 本轮所有请求的结果
  summary: T_RunSummary;        // 本轮迭代的汇总统计
};
```

### 1.2 完整的数据结构层次图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  最外层: Runner 完整执行结果 (T_RunnerResults[])                              │
│  [ 迭代0, 迭代1, 迭代2, ... ]                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                            │
                            │ 每个元素代表一轮迭代
                            ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  单次迭代: T_RunnerResults                                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  iterationIndex: 0                  ←── 当前轮次索引（第 N 行数据）         │
│  iterationData: { id: 1, name: 'x' } ←── CSV/JSON 行数据（整轮共享）       │
│                                                                             │
│  results: [                          ←── 本轮所有请求的执行结果             │
│    请求1结果,                     ←── T_RunnerRequestExecutionResult       │
│    请求2结果,                     ←── T_RunnerRequestExecutionResult       │
│    ...                                                                     │
│  ]                                                                          │
│                                                                             │
│  summary: {                         ←── 本轮迭代的汇总统计                 │
│    totalRequests: 5,                                                         │
│    passedRequests: 4,                                                        │
│    failedRequests: 1,                                                        │
│    ...                                                                       │
│  }                                                                           │
└─────────────────────────────────────────────────────────────────────────────┘
                            │
                            │ 每个请求结果
                            ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  单个请求: T_RunnerRequestExecutionResult                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│  iterationIndex: 0                  ←── 归属轮次索引（同迭代内所有请求相同）│
│  name: "获取用户信息"                                                         │
│  path: "/users/:id"                                                          │
│  request: { method, url, headers, data }                                     │
│  response: { status, statusText, headers, data }                             │
│  status: "pass" / "fail" / "error" / "skipped"                               │
│  assertionResults: [ ... ]                                                   │
│  testResults: [ ... ]                                                        │
│  preRequestTestResults: [ ... ]                                              │
│  postResponseTestResults: [ ... ]                                            │
│  runDuration: 123                                                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. iterationData 归属与汇总链路的关系：证据化说明

### 2.1 核心结论（基于代码证据）

**`iterationData` 属于迭代批次级别，不属于单个请求级别**

### 2.2 证据链

**证据 1: 类型定义层面** (`packages/bruno-common/src/runner/types/index.ts`)
```typescript
// ✅ iterationIndex 同时存在于两个层级:
//   - 单次迭代层: T_RunnerResults.iterationIndex (行 100)
//   - 单个请求层: T_RunnerRequestExecutionResult.iterationIndex (行 85)

// ✅ iterationData 只存在于单次迭代层:
//   - T_RunnerResults.iterationData (行 101)
//   - ❌ T_RunnerRequestExecutionResult 中不存在 iterationData

// 推断: 同一次迭代的所有请求共享同一个 iterationData，数据只在迭代层级存一份
```

**证据 2: 汇总逻辑层面** (`packages/bruno-common/src/runner/runner-summary.ts`)
```typescript
// 汇总函数签名:
export const getRunnerSummary = (results: T_RunnerRequestExecutionResult[]): T_RunSummary

// 调用时机:
// 每一轮迭代的所有请求执行完毕后，
// 将本轮所有请求结果组成的 results 数组传给 getRunnerSummary，
// 计算得到本轮的 summary。

// 最终拼装:
// iterationIndex → 迭代序号
// iterationData → 原始数据行（CSV/JSON 的一行）
// results → 本轮所有请求结果
// summary → getRunnerSummary(results) 的输出
```

### 2.3 为什么 iterationData 不属于单个请求级别？

**设计意图分析**:

```
CSV 数据文件示例:
┌─────────┬─────────┬─────────┐
│ userId  │ token   │ expected│
├─────────┼─────────┼─────────┤
│ 1001    │ abc123  │ 200     │  ←── 迭代 0 的 iterationData
│ 1002    │ def456  │ 200     │  ←── 迭代 1 的 iterationData
│ 1003    │ ghi789  │ 404     │  ←── 迭代 2 的 iterationData
└─────────┴─────────┴─────────┘
     │
     │ 每行数据对应一轮迭代
     ▼
迭代 0 开始
  ├── iterationIndex = 0
  ├── iterationData = { userId: 1001, token: 'abc123', expected: 200 }
  │
  ├── 请求 1 执行: GET /users/{{userId}}
  │     └── iterationIndex = 0 (标记归属)
  │
  ├── 请求 2 执行: POST /auth
  │     └── iterationIndex = 0 (标记归属)
  │
  ├── 请求 3 执行: DELETE /session
  │     └── iterationIndex = 0 (标记归属)
  │
  └── 迭代 0 结束: 调用 getRunnerSummary([请求1, 请求2, 请求3])
          └── summary = 本轮统计结果
```

**关键设计考量**:

1. **避免数据冗余**: 同一迭代的 10 个请求，如果都挂 `iterationData`，数据重复存 10 次
2. **查询效率**: 按 `iterationIndex` 关联查询即可，单个请求只需要索引标记
3. **语义正确**: `iterationData` 描述的是**迭代**的输入条件，不是请求本身的属性
4. **汇总方便**: 每轮迭代结束直接汇总，结果结构天然对齐输入数据行

### 2.4 完整的汇总链路时序

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              数据驱动执行总览                                │
└─────────────────────────────────────────────────────────────────────────────┘

阶段 1: 解析数据文件
  ├── 读取 CSV/JSON 文件
  ├── 解析为数组: dataRows = [row0, row1, row2, ...]
  ├── 迭代总数 = dataRows.length

阶段 2: 外层循环（遍历数据行）
  for (let i = 0; i < dataRows.length; i++) {
    const iterationData = dataRows[i];
    
    // 注入数据到 runtimeVariables
    runtimeVariables['$iteration'] = i;
    Object.assign(runtimeVariables, iterationData);
    
    // 初始化当前迭代的请求结果数组
    const currentIterationResults = [];

阶段 3: 内层循环（执行请求集合）
    let currentRequestIndex = 0;
    while (currentRequestIndex < folderRequests.length) {
      // 执行单个请求 ...
      
      // 收集单个请求结果
      const requestResult = {
        iterationIndex: i,  // ✅ 标记归属的迭代索引
        name: '...',
        path: '...',
        request: { ... },
        response: { ... },
        status: 'pass',
        // ... 其他字段
      };
      
      currentIterationResults.push(requestResult);
      currentRequestIndex++;
    }

阶段 4: 本轮迭代汇总
    // 调用汇总函数
    const iterationSummary = getRunnerSummary(currentIterationResults);
    
    // 拼装完整的迭代结果
    const fullIterationResult = {
      iterationIndex: i,          // 迭代索引
      iterationData: iterationData, // 原始数据行 ✅ iterationData 在此处！
      results: currentIterationResults,  // 本轮所有请求结果
      summary: iterationSummary   // 本轮汇总统计
    };
    
    // 加入最终结果数组
    finalResults.push(fullIterationResult);
  }

阶段 5: 所有迭代完成
  返回 finalResults 给前端渲染
```

### 2.5 数据溯源关系图

```
CSV/JSON 数据行
     │
     ├── iterationIndex (注入到 runtimeVariables) → 每个请求结果记录 iterationIndex
     ├── iterationData (注入到 runtimeVariables) → 仅迭代结果保留原始数据
     │
     ▼
T_RunnerResults (迭代层)
  ├── iterationIndex: 0
  ├── iterationData: { userId: 1001, token: 'abc123' }  ←── 原始数据唯一存储点
  │
  ├── results: [请求1, 请求2, 请求3]
  │       ├── 请求1: { iterationIndex: 0, ... }  ←── 通过索引关联
  │       ├── 请求2: { iterationIndex: 0, ... }  ←── 通过索引关联
  │       └── 请求3: { iterationIndex: 0, ... }  ←── 通过索引关联
  │
  └── summary: { ... }  ←── getRunnerSummary(results) 的输出
```

---

## 3. 数据驱动迭代数据进入执行链机制（修正版）

### 3.1 当前实现状态

基于类型定义分析，Bruno 的数据驱动功能设计已经完整：

| 特性 | 类型定义状态 | 实现状态 |
|------|------------|---------|
| 迭代索引标记 (iterationIndex) | ✅ 已定义（请求层 + 迭代层） | 待实现 |
| 迭代数据存储 (iterationData) | ✅ 已定义（迭代层） | 待实现 |
| 单个请求结果结构 | ✅ 已定义 | 部分实现 |
| 迭代级汇总 (summary) | ✅ 已定义 | ✅ 已实现 (getRunnerSummary) |
| 多轮迭代结果数组 | ✅ 已定义 (T_RunnerResults[]) | 待实现 |

### 3.2 预期的数据驱动完整实现流程

```
数据文件 (CSV/JSON)
     │
     ▼
┌──────────────────────────────────────────────────────────┐
│  数据解析层                                               │
│  - CSV: Papa.parse(csvFile) → dataRows[]                │
│  - JSON: JSON.parse(jsonFile) → dataRows[]              │
└──────────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────┐
│  外层循环 (迭代层)                                        │
│  for (i = 0; i < dataRows.length; i++) {                 │
│    - 注入 iterationData = dataRows[i] 到 runtimeVariables│
│    - 执行内层请求循环                                      │
│    - 收集所有请求结果                                      │
│    - 调用 getRunnerSummary 得到本轮 summary              │
│    - 拼装 T_RunnerResults 对象                            │
│  }                                                        │
└──────────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────┐
│  内层循环 (请求层)                                        │
│  while (currentRequestIndex < folderRequests.length) {   │
│    - 执行单个请求                                          │
│    - 记录 iterationIndex 到请求结果                        │
│    - 正常执行流程 (Pre-request → HTTP → Post-response...)│
│  }                                                        │
└──────────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────┐
│  返回 T_RunnerResults[] 给前端                            │
│  前端渲染:                                                │
│    - 按迭代展开 / 折叠                                    │
│    - 显示原始 iterationData (CSV 行预览)                  │
│    - 显示每轮的 summary 统计                              │
│    - 显示每轮的每个请求详情                                │
└──────────────────────────────────────────────────────────┘
```

---

## 4. 跨请求变量传递机制（保持不变）

### 4.1 核心实现原理

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

### 4.2 变量作用域与生命周期（保持不变）

| 变量类型 | 存储位置 | 生命周期 | 跨请求传递 |
|---------|---------|---------|-----------|
| **Runtime Variables** | `runtimeVariables` | Runner 执行全程 | ✅ 支持 |
| **Environment Variables** | `envVariables` | Runner 执行全程 | ✅ 支持 (需 `{persist: true}` 才持久化到文件) |
| **Collection Variables** | `collectionVariables` | 单次请求 | ❌ 不支持 (只读) |
| **Folder Variables** | `folderVariables` | 单次请求 | ❌ 不支持 (只读) |
| **Request Variables** | `requestVariables` | 单次请求 | ❌ 不支持 (只读) |
| **Global Env Variables** | `globalEnvironmentVariables` | Runner 执行全程 | ✅ 支持 (部分实现) |

---

## 5. Pre-request 场景下 stopExecution 生效时序（保持不变）

### 5.1 关键结论

**`bru.stopExecution()` 在 Pre-request 中调用时：不会立即中断当前请求！**

1. 只会设置标志位 `stopRunnerExecution = true`
2. 当前请求会完整执行到底（HTTP 发送 → Post-response → Assertions → Tests）
3. 真正中断发生在当前请求全部执行完毕后，才跳出 while 循环

### 5.2 时序对比

```
bru.skipRequest():
  Pre-request ──┤ 调用 skipRequest() ──▶ 立即 continue → 跳过当前请求 → 执行下一个

bru.stopExecution():
  Pre-request ──┤ 调用 stopExecution() ──▶ 设置标志 → 继续执行
                                                ↓
                                    发送 HTTP 请求
                                                ↓
                                    Post-response 脚本
                                                ↓
                                    Assertions 断言
                                                ↓
                                    Tests 测试脚本
                                                ↓
                                    检查标志 → break 循环 → 终止 Runner
```

---

## 6. 总结

### 6.1 类型与数据结构校正要点

| 字段 | 所属层级 | 作用 |
|------|---------|------|
| `iterationIndex` (请求层) | T_RunnerRequestExecutionResult | 标记请求属于哪一轮迭代 |
| `iterationIndex` (迭代层) | T_RunnerResults | 当前迭代的序号 |
| `iterationData` | T_RunnerResults | 原始 CSV/JSON 数据行（仅迭代层级存储） |
| `results` | T_RunnerResults | 本轮迭代所有请求的执行结果数组 |
| `summary` | T_RunnerResults | `getRunnerSummary(results)` 的输出，本轮统计 |

### 6.2 iterationData 归属设计的合理性

✅ **避免冗余**: 同一迭代的所有请求共享数据行，只存储一次  
✅ **语义正确**: iterationData 是迭代的输入，不是请求的属性  
✅ **查询高效**: 通过 iterationIndex 关联即可溯源  
✅ **汇总方便**: 每轮迭代结束直接汇总，结构天然对齐输入数据

### 6.3 其他保持不变的结论

- 跨请求变量传递通过 `runtimeVariables` 对象引用共享实现
- Pre-request 中 `stopExecution()` 不会立即中止，当前请求会完整执行到底
- `skipRequest()` 是立即跳过当前请求，`stopExecution()` 是完成当前后终止

---

## 7. 版本变更记录

| 版本 | 主要变更 | 日期 |
|------|---------|------|
| R1 | 完整流程分析 | 2026-05-17 |
| R2 | 补充数据驱动、跨请求传递、stopExecution 时序 | 2026-05-17 |
| **R3** | **✓ 校正 iterationData 归属错误（请求层 → 迭代层）**<br>✓ 补充证据化说明：归属与汇总链路的关系 | **2026-05-17** |

**报告版本**: R3  
**分析日期**: 2026-05-17  
**代码范围**: packages/bruno-common, packages/bruno-electron, packages/bruno-js  
**核心证据文件**: 
- `packages/bruno-common/src/runner/types/index.ts`
- `packages/bruno-common/src/runner/runner-summary.ts`
