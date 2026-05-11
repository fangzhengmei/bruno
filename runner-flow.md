# Bruno Collection Runner 执行流程分析

## 目录
1. [整体架构概览](#整体架构概览)
2. [数据表批量参数化支持情况](#数据表批量参数化支持情况)
3. [CLI Runner 实现位置](#cli-runner-实现位置)
4. [桌面 Runner 实现位置](#桌面-runner-实现位置)
5. [请求执行顺序与跳转时序](#请求执行顺序与跳转时序)
6. [迭代变量注入机制](#迭代变量注入机制)
7. [失败处理策略与分支时序](#失败处理策略与分支时序)
8. [核心代码位置汇总](#核心代码位置汇总)
9. [文档与实现不一致说明](#文档与实现不一致说明)

---

## 整体架构概览

Bruno 的 Collection Runner 有两套独立实现，共享核心逻辑一致：

| 版本 | 核心执行文件 | 循环起始行号 |
|-----|-------------|----------|
| **CLI 版本** | `bruno-cli/src/commands/run.js` | `line 680-765 |
| **桌面端版本** | `bruno-electron/src/ipc/network/index.js` | `line 1368-1893 |

### 主要组件：
- **触发入口**: `bruno-app/src/components/Sidebar/Collections/Collection/CollectionItem/RunCollectionItem/index.js`
- **Runner 结果面板**: `bruno-app/src/components/RunnerResults/index.jsx`
- **运行配置面板**: `bruno-app/src/components/RunnerResults/RunConfigurationPanel/index.jsx`
- **Action 层**: `bruno-app/src/providers/ReduxStore/slices/collections/actions.js`
- **单请求执行 (CLI)**: `bruno-cli/src/runner/run-single-request.js`
- **单请求执行 (桌面端)**: `bruno-electron/src/ipc/network/index.js:runRequest()`

---

## 数据表批量参数化支持情况

### ⚠️ 关键发现：类型已预留，功能未实现

#### 类型定义 (`bruno-common/src/runner/types/index.ts:99-104`)
```typescript
export type T_RunnerResults = {
  iterationIndex: number,
  iterationData?: any, // todo - csv/json row data
  results: T_RunnerRequestExecutionResult[],
  summary: T_RunSummary
};
```

**代码注释明确标注**: `todo - csv/json row data`

### 桌面端 Runner 现状
**位置**: `bruno-app/src/components/RunnerResults/RunConfigurationPanel/index.jsx`

当前支持的配置功能：
| 功能 | 支持状态 | 说明 |
|-----|---------|------|
| 🔘 请求多选 | ✅ 已实现 | 支持选中/取消选中单个请求 |
| 🔘 全选/取消全选 | ✅ 已实现 | 批量选择所有请求 |
| 🔀 拖拽排序 | ✅ 已实现 | 支持拖拽调整执行顺序 |
| 🏷️ 标签过滤 | ✅ 已实现 | 按 Include/Exclude 标签过滤 |
| ⏱️ 延迟设置 | ✅ 已实现 | 支持请求间延迟 (ms) |
| 🔄 重置配置 | ✅ 已实现 | 恢复默认选择和顺序 |
| 📊 数据文件 (CSV/JSON) | ❌ 未实现 | UI 无入口，代码无逻辑 |
| 🔁 迭代执行 | ❌ 未实现 | 外层无迭代循环 |

**UI 入口代码**:
```javascript
// bruno-app/src/components/RunnerResults/index.jsx:255-270
<div className="runner-section-title mt-6">Timings</div>
<div className="runner-section mt-2">
  <label>Delay between requests (ms)</label>
  <input
    type="number"
    className="block textbox w-full mt-2"
    placeholder="e.g. 5"
    value={delay}
    onChange={(e) => setDelay(e.target.value)}
  />
</div>

// 无数据文件选择 UI
```

### CLI Runner 现状
**位置**: `bruno-cli/src/commands/run.js`

当前支持的命令行参数：
```bash
# 已实现的参数
bruno run folder -r                   # 递归执行
bruno run folder --delay 1000         # 请求延迟 (ms)
bruno run folder --tags smoke         # 标签过滤
bruno run folder --exclude-tags skip  # 排除标签
bruno run folder --env production     # 环境选择
bruno run folder --env-var key=value  # 变量覆盖
bruno run folder --bail               # 遇错即停

# 未实现的参数
# --data-file / -D (无此参数定义)
# --iteration-count / -I (无此参数定义)
# --iterations (无此参数定义)
```

**CLI 循环实现**:
```javascript
// bruno-cli/src/commands/run.js:680-765
let currentRequestIndex = 0;
let nJumps = 0; // count the number of jumps to avoid infinite loops
while (currentRequestIndex < requestItems.length) {
  // 仅请求级循环，无外层数据迭代循环
  // line 746-764: 跳转逻辑
}
```

**桌面端循环实现**:
```javascript
// bruno-electron/src/ipc/network/index.js:1368-1893
let currentRequestIndex = 0;
let nJumps = 0; // count the number of jumps to avoid infinite loops
while (currentRequestIndex < folderRequests.length) {
  // 仅请求级循环，无外层数据迭代循环
  // line 1875-1892: 跳转逻辑
}
```

### 结论：数据表批量参数化尚未实现
1. ✅ **类型预留**: `iterationIndex`、`iterationData` 类型已定义
2. ❌ **UI 缺失**: 桌面端无数据文件选择界面
3. ❌ **CLI 参数缺失**: 命令行无 `--data-file` 类参数
4. ❌ **执行逻辑缺失**: 两套 Runner 的核心循环均无外层迭代逻辑
5. ❌ **变量注入缺失**: 无迭代数据注入变量的逻辑

---

## CLI Runner 实现位置

### CLI 核心循环
**文件**: `bruno-cli/src/commands/run.js`
**行号**: `line 680-765`

```javascript
// line 680-681: 循环初始化
let currentRequestIndex = 0;
let nJumps = 0; // count the number of jumps to avoid infinite loops

// line 682: while 循环开始
while (currentRequestIndex < requestItems.length) {

  // line 687-699: 执行单个请求
  const result = await runSingleRequest(...);

  // line 703-706: 延迟处理
  if (isValidDelay && !isLastRun) {
    await new Promise((resolve) => setTimeout(resolve, delay));
  }

  // line 727-737: --bail 终止逻辑
  if (bail) { /* ... */ break; }

  // line 740: 获取下一个请求名
  const nextRequestName = result?.nextRequestName;

  // line 742-744: stopRunner 终止
  if (result?.shouldStopRunnerExecution) {
    break;
  }

  // line 746-764: 跳转决策
  if (nextRequestName !== undefined) {
    nJumps++;
    if (nJumps > 10000) { /* 无限循环防护 */ }
    if (nextRequestName === null) { break; }
    const nextRequestIdx = requestItems.findIndex(...);
    if (nextRequestIdx >= 0) {
      currentRequestIndex = nextRequestIdx; // 跳转到指定请求
    } else {
      currentRequestIndex++; // 顺序执行
    }
  } else {
    currentRequestIndex++; // 顺序执行
  }
}
```

### CLI 单请求执行
**文件**: `bruno-cli/src/runner/run-single-request.js`
**职责**: 执行单个 HTTP 请求，包括脚本执行，变量插值

---

## 桌面 Runner 实现位置

### 桌面端核心循环
**文件**: `bruno-electron/src/ipc/network/index.js`
**行号**: `line 1368-1893`

```javascript
// line 1368-1369: 循环初始化
let currentRequestIndex = 0;
let nJumps = 0; // count the number of jumps to avoid infinite loops

// line 1370: while 循环开始
while (currentRequestIndex < folderRequests.length) {

  // line 1372-1376: 检查取消信号
  if (abortController.signal.aborted) { throw error; }

  // line 1397-1413: 跳过 gRPC 请求
  if (item.type === 'grpc-request') { /* ... */ continue; }

  // line 1420-1439: 跳过含 Prompt 变量的请求
  if (promptVars.length > 0) { /* ... */ continue; }

  // line 1456-1505: Pre-request 脚本执行
  // line 1515-1530: skipRequest 跳过当前请求

  // line 1532-1694: 发送 HTTP 请求

  // line 1697-1754: Post-response 脚本执行

  // line 1756-1775: 执行 Assertions

  // line 1777-1853: 执行 Tests 脚本

  // line 1863-1873: stopRunnerExecution 终止
  if (stopRunnerExecution) { /* ... */ break; }

  // line 1875-1892: 跳转决策
  if (nextRequestName !== undefined) {
    nJumps++;
    if (nJumps > 10000) { throw error; }
    if (nextRequestName === null) { break; }
    const nextRequestIdx = folderRequests.findIndex(...);
    if (nextRequestIdx >= 0) {
      currentRequestIndex = nextRequestIdx; // 跳转到指定请求
    } else {
      currentRequestIndex++; // 顺序执行
    }
  } else {
    currentRequestIndex++; // 顺序执行
  }
}
```

### 桌面端 IPC 通信层
**文件**: `bruno-electron/src/ipc/network/index.js`
**行号**: `line 1273-1914` (renderer:run-collection-folder handler

### 桌面端 UI 触发层
**文件**: `bruno-app/src/components/RunnerResults/RunConfigurationPanel/index.jsx`
**职责**: 请求选择、排序、标签过滤配置

---

## 请求执行顺序与跳转时序

### 1. 初始化阶段 (Collection Runner 启动)

**桌面端执行流程**：
```
用户点击 Run 按钮
    ↓
打开 RunCollectionItem 弹窗
    ↓
用户选择 Run / Recursive Run，设置延迟
    ↓
runCollectionFolder() action 被调用
    ↓
RunnerResults 面板打开
    ↓
RunConfigurationPanel 加载请求列表
    ↓
根据 savedConfiguration 恢复选中状态和排序
    ↓
通过 IPC 调用 renderer:run-collection-folder (bruno-electron/src/ipc/network/index.js:1273)
    ↓
创建取消令牌 (cancelTokenUid)
    ↓
发送 'testrun-started' 事件到 UI
```

**CLI 执行流程**：
```
用户执行 bruno run 命令 (bruno-cli/src/commands/run.js:309)
    ↓
解析命令行参数 (--delay, --tags, --env 等)
    ↓
加载 collection 和环境配置
    ↓
收集请求列表 (递归 / 非递归)
    ↓
应用标签过滤
    ↓
开始执行请求循环 (line 680)
```

### 2. 请求收集与排序

**排序优先级**（从高到低）：
1. **用户自定义排序**: `selectedRequestUids` 指定的顺序
2. **保存的配置排序**: `runnerConfiguration.requestItemsOrder`
3. **文件夹 seq 属性排序**: `sortByNameThenSequence()`
4. **递归模式**: 深度优先遍历

```javascript
// 桌面端: bruno-electron/src/ipc/network/index.js:1352-1366
// CLI: bruno-cli/src/commands/run.js:620-639 (无 selectedRequestUids 过滤
```

### 3. 核心循环与跳转控制时序

**桌面端精确执行时序** (`bruno-electron/src/ipc/network/index.js:1368-1893):

```
  ┌─────────────────────────────────────────────────────────────┐
  │  WHILE (currentRequestIndex < folderRequests.length)        │
  └─────────────────────────────────────────────────────────────┘
    ↓ 1372-1376
  ┌─────────────────────────────────────────────────────────────┐
  │  ✅ 检查取消信号 (abortController.signal.aborted)          │
  │     → 抛出错误，终止整个 Runner                             │
  └─────────────────────────────────────────────────────────────┘
    ↓ 1378
  ┌─────────────────────────────────────────────────────────────┐
  │  🔄 重置 stopRunnerExecution = false                        │
  └─────────────────────────────────────────────────────────────┘
    ↓ 1380-1395
  ┌─────────────────────────────────────────────────────────────┐
  │  📨 发送 'request-queued' 事件                              │
  └─────────────────────────────────────────────────────────────┘
    ↓ 1398-1413
  ┌─────────────────────────────────────────────────────────────┐
  │  ⏭️  跳过 gRPC 请求 (type === 'grpc-request')               │
  │     → 发送 'runner-request-skipped'                         │
  │     → currentRequestIndex++                                 │
  │     → continue 下一轮循环                                    │
  └─────────────────────────────────────────────────────────────┘
    ↓ 1415
  ┌─────────────────────────────────────────────────────────────┐
  │  📦 prepareRequest() 合并变量                               │
  └─────────────────────────────────────────────────────────────┘
    ↓ 1420-1439
  ┌─────────────────────────────────────────────────────────────┐
  │  ⏭️  跳过含 Prompt 变量的请求                               │
  │     → 发送 'runner-request-skipped'                         │
  │     → currentRequestIndex++                                 │
  │     → continue 下一轮循环                                    │
  └─────────────────────────────────────────────────────────────┘
    ↓ 1441
  ┌─────────────────────────────────────────────────────────────┐
  │  try { 开启请求级 try-catch                                  │
  └─────────────────────────────────────────────────────────────┘
    ↓ 1456-1505
  ┌─────────────────────────────────────────────────────────────┐
  │  🔴 Pre-request 脚本执行                                    │
  │     → 捕获错误，提取 partialResults                         │
  │     → appendScriptErrorResult() 追加错误结果                │
  │     → 发送 'test-results-pre-request'                       │
  │     → 发送脚本执行通知                                       │
  │     → 有错误 → throw → 进入 catch                           │
  └─────────────────────────────────────────────────────────────┘
    ↓ 1507-1509
  ┌─────────────────────────────────────────────────────────────┐
  │  🎯 跳转检查 1: preRequestScriptResult.nextRequestName     │
  │     → 设置 nextRequestName 变量                             │
  └─────────────────────────────────────────────────────────────┘
    ↓ 1511-1513
  ┌─────────────────────────────────────────────────────────────┐
  │  🛑 终止检查 1: preRequestScriptResult.stopExecution        │
  │     → 设置 stopRunnerExecution = true                       │
  └─────────────────────────────────────────────────────────────┘
    ↓ 1515-1530
  ┌─────────────────────────────────────────────────────────────┐
  │  ⏭️  跳过检查: preRequestScriptResult.skipRequest           │
  │     → 发送 'runner-request-skipped'                         │
  │     → currentRequestIndex++                                 │
  │     → continue 下一轮循环                                    │
  └─────────────────────────────────────────────────────────────┘
    ↓ 1532-1694
  ┌─────────────────────────────────────────────────────────────┐
  │  📤 发送 HTTP 请求 (axios)                                   │
  │     → 应用 delay 延迟                                       │
  │     → 成功 → 解析响应数据                                   │
  │     → 4XX/5XX → 继续执行后续流程 (Post-response, Tests)    │
  │     → 网络错误/DNS错误 → executeRequestOnFailHandler()      │
  │     → throw error → 进入 catch                              │
  └─────────────────────────────────────────────────────────────┘
    ↓ 1697-1754
  ┌─────────────────────────────────────────────────────────────┐
  │  🟢 Post-response 脚本执行                                  │
  │     → 捕获错误，提取 partialResults                         │
  │     → appendScriptErrorResult() 追加错误结果                │
  │     → 发送脚本执行通知                                       │
  └─────────────────────────────────────────────────────────────┘
    ↓ 1739-1741
  ┌─────────────────────────────────────────────────────────────┐
  │  🎯 跳转检查 2: postResponseScriptResult.nextRequestName    │
  │     → 覆盖 nextRequestName 变量                             │
  └─────────────────────────────────────────────────────────────┘
    ↓ 1743-1745
  ┌─────────────────────────────────────────────────────────────┐
  │  🛑 终止检查 2: postResponseScriptResult.stopExecution      │
  │     → 设置 stopRunnerExecution = true                       │
  └─────────────────────────────────────────────────────────────┘
    ↓ 1748-1754
  ┌─────────────────────────────────────────────────────────────┐
  │  📨 发送 'test-results-post-response' 事件                  │
  └─────────────────────────────────────────────────────────────┘
    ↓ 1756-1775
  ┌─────────────────────────────────────────────────────────────┐
  │  ✅ 执行 Assertions → 记录结果                                      │
  └─────────────────────────────────────────────────────────────┘
    ↓ 1777-1853
  ┌─────────────────────────────────────────────────────────────┐
  │  🧪 执行 Tests 脚本                                                │
  │     → 捕获错误，提取 partialResults                         │
  │     → appendScriptErrorResult() 追加错误结果                │
  └─────────────────────────────────────────────────────────────┘
    ↓ 1817-1819
  ┌─────────────────────────────────────────────────────────────┐
  │  🎯 跳转检查 3: testResults.nextRequestName                 │
  │     → 覆盖 nextRequestName 变量 (最终生效)                  │
  └─────────────────────────────────────────────────────────────┘
    ↓ 1821-1837
  ┌─────────────────────────────────────────────────────────────┐
  │  📨 发送 'test-results' 事件                                │
  │  📨 发送 'script-environment-update' 更新变量                │
  └─────────────────────────────────────────────────────────────┘
    ↓ 1854-1861
  ┌─────────────────────────────────────────────────────────────┐
  │  } catch (error) { 捕获请求级错误                           │
  │     → 发送 'error' 事件                                     │
  └─────────────────────────────────────────────────────────────┘
    ↓ 1863-1873
  ┌─────────────────────────────────────────────────────────────┐
  │  🛑 Runner 终止检查: stopRunnerExecution                    │
  │     → 发送 'testrun-ended' (statusText: terminated)        │
  │     → break 退出循环                                        │
  └─────────────────────────────────────────────────────────────┘
    ↓ 1875-1892
  ┌─────────────────────────────────────────────────────────────┐
  │  🎯 跳转决策                                                │
  │     ├─ nextRequestName === null → break 终止                │
  │     ├─ nextRequestName 存在且找到 → 跳转到指定 index        │
  │     ├─ nextRequestName 不存在找不到 → index++ 顺序执行      │
  │     └─ 未设置 nextRequestName → index++ 顺序执行            │
  └─────────────────────────────────────────────────────────────┘
    ↓ 循环结束
  ┌─────────────────────────────────────────────────────────────┐
  │  📨 发送 'testrun-ended' 事件 (正常结束)                    │
  └─────────────────────────────────────────────────────────────┘
```

**CLI 精确执行时序** (`bruno-cli/src/commands/run.js:680-765`):

```
  ┌─────────────────────────────────────────────────────────────┐
  │  WHILE (currentRequestIndex < requestItems.length)        │
  └─────────────────────────────────────────────────────────────┘
    ↓ 687-699
  ┌─────────────────────────────────────────────────────────────┐
  │  📦 runSingleRequest() 执行单个请求                        │
  └─────────────────────────────────────────────────────────────┘
    ↓ 703-706
  ┌─────────────────────────────────────────────────────────────┐
  │  ⏱️ 延迟处理 (--delay 参数                                  │
  └─────────────────────────────────────────────────────────────┘
    ↓ 712-718
  ┌─────────────────────────────────────────────────────────────┐
  │  📊 收集执行结果                                        │
  └─────────────────────────────────────────────────────────────┘
    ↓ 727-737
  ┌─────────────────────────────────────────────────────────────┐
  │  🛑 --bail 终止检查 (遇错即停)                              │
  │     → 有失败 → break 终止                                  │
  └─────────────────────────────────────────────────────────────┘
    ↓ 740
  ┌─────────────────────────────────────────────────────────────┐
  │  🎯 获取 nextRequestName                                    │
  └─────────────────────────────────────────────────────────────┘
    ↓ 742-744
  ┌─────────────────────────────────────────────────────────────┐
  │  🛑 shouldStopRunnerExecution 终止                            │
  │     → break 退出循环                                        │
  └─────────────────────────────────────────────────────────────┘
    ↓ 746-764
  ┌─────────────────────────────────────────────────────────────┐
  │  🎯 跳转决策                                                │
  │     ├─ nextRequestName === null → break 终止                │
  │     ├─ nextRequestName 存在且找到 → 跳转到指定 index        │
  │     ├─ nextRequestName 不存在找不到 → index++ 顺序执行      │
  │     └─ 未设置 nextRequestName → index++ 顺序执行            │
  └─────────────────────────────────────────────────────────────┘
```

### 跳转优先级与生效顺序

**跳转设置优先级**（后面的覆盖前面的）：
1. Pre-request 脚本设置 → 最先设置，可能被覆盖
2. Post-response 脚本设置 → 覆盖 Pre-request 的设置
3. Tests 脚本设置 → 最后设置，最终生效

**跳转 API**：
```javascript
// 跳转到指定请求名称
bru.setNextRequest('Request Name');

// 终止执行
bru.setNextRequest(null);
bru.stopRunner(); // 等价于 setNextRequest(null)
```

**跳转查找逻辑**：
- **桌面端**: `bruno-electron/src/ipc/network/index.js:1883
- **CLI**: `bruno-cli/src/commands/run.js:755`

```javascript
const nextRequestIdx = folderRequests.findIndex((request) => request.name === nextRequestName);
```
- **按名称匹配**: `request.name` 完全匹配（大小写敏感）
- **未找到**: 输出 `console.error`，顺序执行下一个

**无限循环防护**：
- **桌面端**: `line 1877`
- **CLI**: `line 747-751`

```javascript
nJumps++;
if (nJumps > 10000) {
    throw new Error('Too many jumps, possible infinite loop');
}
```

---

## 迭代变量注入机制

### 变量层级结构

Runner 采用**多层级变量覆盖**机制，优先级从高到低：

```
Runtime Variables (运行时) ← 脚本中 bru.setVar() 设置
    ↑
Request Variables (请求级) ← Pre-request 脚本设置
    ↑
Folder Variables (文件夹级) ← prepareRequest() 合并
    ↑
Environment Variables (环境变量) ← 环境文件配置
    ↑
Collection Variables (集合级) ← 集合配置
    ↑
Global Environment Variables (全局环境) ← 全局环境配置
    ↑
Process Environment Variables (系统环境) ← 进程环境变量
```

### 变量注入时机（精确位置）

#### 1. 准备阶段 - `prepareRequest()`
**桌面端位置**: `bruno-electron/src/ipc/network/prepare-request.js`
**CLI 位置**: `bruno-cli/src/runner/prepare-request.js`
- 合并 Collection → Folder → Request 三级变量
- 不执行变量插值，仅合并变量对象

#### 2. Pre-request 脚本执行后
**桌面端位置**: `bruno-electron/src/ipc/network/index.js:runPreRequest()`
**CLI 位置**: `bruno-cli/src/runner/run-single-request.js`
- 脚本中通过 `bru.setEnvVar()` / `bru.setVar()` 修改变量
- **桌面端**: 执行完成后发送 `script-environment-update` 事件同步到 UI
- **注意**: 此阶段修改的变量会影响当前请求的插值

#### 3. 变量插值 - `interpolateVars()`
**桌面端位置**: `bruno-electron/src/ipc/network/interpolate-vars.js`
**CLI 位置**: `bruno-cli/src/runner/interpolate-vars.js`
- 执行变量替换：`{{variableName}}` → 实际值
- 支持嵌套对象访问：`{{user.name}}`
- 支持变量引用其他变量（递归解析）

#### 4. Post-response 脚本执行后
**桌面端位置**: `bruno-electron/src/ipc/network/index.js:runPostResponse()`
**CLI 位置**: `bruno-cli/src/runner/run-single-request.js`
- 脚本中修改的变量影响**后续请求**，不影响当前请求

#### 5. Tests 脚本执行后
**桌面端位置**: `bruno-electron/src/ipc/network/index.js:1827-1837`
**CLI 位置**: `bruno-cli/src/runner/run-single-request.js`
- 最后一次变量更新机会
- 桌面端发送 `script-environment-update` 同步到 UI
- 影响后续所有请求

### 变量操作 API

| API | 作用域 | 生效时机 | 持久化 |
|-----|--------|---------|-------|
| `bru.getEnvVar('name')` | 环境变量 | 立即 | ✅ 持久化到环境 |
| `bru.setEnvVar('name', 'value')` | 环境变量 | 后续请求 | ✅ 持久化到环境 |
| `bru.setVar('name', 'value')` | 运行时变量 | 后续请求 | ❌ 仅内存 |
| `bru.getVar('name')` | 运行时变量 | 立即 | ❌ 仅内存 |

---

## 失败处理策略与分支时序

### 失败分支总览

```
请求执行过程中可能的失败点：
    ├─ Pre-request 脚本语法错误 / 运行时错误
    ├─ HTTP 请求网络错误 (DNS, 连接超时, SSL 等)
    ├─ HTTP 4XX/5XX 响应 (有响应体，继续执行)
    ├─ Post-response 脚本错误
    ├─ Assertions 断言失败
    └─ Tests 脚本错误 / 断言失败
```

### 1. Pre-request 脚本错误分支

**桌面端精确时序** (`bruno-electron/src/ipc/network/index.js:1456-1505`):

```
try { runPreRequest() }
    ↓
catch (error) {
    1. 记录 console.error
    2. preRequestError = error
}
    ↓
检查 preRequestError.partialResults
    ↓ 存在
preRequestScriptResult = partialResults
    ↓
preRequestScriptResult = appendScriptErrorResult()
    ↓
发送 'test-results-pre-request' 事件
    ↓
发送脚本执行通知 (含错误信息)
    ↓
if (preRequestError) {
    throw preRequestError  → 进入请求 catch
}
```

**CLI 精确时序** (`bruno-cli/src/runner/run-single-request.js`):
- 逻辑与桌面端一致

**appendScriptErrorResult 逻辑** (`bruno-electron/src/ipc/network/index.js:471-502`):
```javascript
{
    status: 'fail',
    description: 'Pre-Request Script Error',
    error: error.message,
    isScriptError: true  // 标记为脚本执行错误（非断言失败）
}
```

### 2. HTTP 请求错误分支

**分支 1: 用户取消请求**
- **桌面端**: `bruno-electron/src/ipc/network/index.js:1652-1654`
- **CLI**: 无 UI 取消功能

```javascript
if (axios.isCancel(error)) {
    throw error;  // 直接抛出，终止当前请求
}
```

**分支 2: 服务器返回 4XX/5XX**
- **桌面端**: `bruno-electron/src/ipc/network/index.js:1656-1689`
- **CLI**: `bruno-cli/src/runner/run-single-request.js`

```
有 error.response 对象
    ↓
解析响应数据 (promisifyStream)
    ↓
构建 response 对象 (含 duration, timeline 等)
    ↓
✅ 继续执行后续流程：
    ├─ Post-response 脚本
    ├─ Assertions
    └─ Tests 脚本
```

**分支 3: 网络错误 / DNS 错误**
- **桌面端**: `bruno-electron/src/ipc/network/index.js:1690-1693`
- **CLI**: `bruno-cli/src/runner/run-single-request.js`

```
无 error.response 对象
    ↓
executeRequestOnFailHandler() 执行失败回调
    ↓
throw error → 终止当前请求
```

### 3. Post-response 脚本错误分支

**桌面端精确时序** (`bruno-electron/src/ipc/network/index.js:1697-1754`):
```
try { runPostResponse() }
    ↓
catch (error) {
    1. 记录 console.error
    2. postResponseError = error
}
    ↓
检查 postResponseError.partialResults
    ↓ 存在
postResponseScriptResult = partialResults
    ↓
postResponseScriptResult = appendScriptErrorResult()
    ↓
发送脚本执行通知 (含错误信息)
    ↓
✅ 继续执行 Assertions 和 Tests
```

**关键差异**: Post-response 脚本错误**不会终止**当前请求的后续流程

### 4. Assertions 断言失败分支

**位置**: `bruno-electron/src/ipc/network/index.js:1756-1775` (桌面端)
**位置**: `bruno-cli/src/runner/run-single-request.js` (CLI)
- 断言失败仅记录到 `assertionResults`
- 不会抛出异常
- 不会终止 Tests 脚本执行
- Runner 不会停止，继续执行下一个请求

### 5. Tests 脚本错误分支

**桌面端精确时序** (`bruno-electron/src/ipc/network/index.js:1798-1813`):
```
try { testRuntime.runTests() }
    ↓
catch (error) {
    testError = error
    if (error.partialResults) {
        testResults = error.partialResults  // 保留已通过的测试
    } else {
        testResults = { results: [], ... }
    }
}
    ↓
testResults = appendScriptErrorResult()
    ↓
发送 'test-results' 事件
    ↓
✅ 继续下一个请求
```

### 6. Runner 级终止分支

**终止触发条件**：

| 触发方式 | 桌面端位置 | CLI 位置 | 状态文本 |
|---------|---------|---------|---------|
| `bru.stopRunner()` | `line 1863` | `line 742-744` | `collection run was terminated!` (仅桌面端)
| `bru.setNextRequest(null)` | `line 1880` | `line 752` | break 终止
| 用户点击 Cancel | UI 操作 | 无 | 无状态文本
| 跳转超过 10000 次 | `line 1877` | `line 747-751` | 抛出异常
| `--bail` 遇错即停 | 无 | `line 727-737` | break 终止

---

## 核心代码位置汇总

| 功能模块 | 文件路径 | 桌面端行号 | CLI 行号 |
|---------|---------|---------|---------|
| Runner 主循环 | `bruno-electron/src/ipc/network/index.js` | 1368-1893 | - |
| Runner 主循环 (CLI) | `bruno-cli/src/commands/run.js` | - | 680-765 |
| 请求跳转逻辑 | `同上` | 1875-1892 | 746-764 |
| 请求跳转逻辑 (CLI) | `同上` | - | 746-764 |
| Pre-request 执行 | `同上` | 1456-1505 | run-single-request.js |
| Post-response 执行 | `同上` | 1697-1754 | run-single-request.js |
| HTTP 请求错误处理 | `同上` | 1650-1694 | run-single-request.js |
| 错误结果追加 | `同上` | 471-502 | - |
| Runner 配置面板 | `bruno-app/src/components/RunnerResults/RunConfigurationPanel/index.jsx` | 1-450 | - |
| Runner 结果面板 | `bruno-app/src/components/RunnerResults/index.jsx` | 1-566 | - |
| CLI 命令定义 | `bruno-cli/src/commands/run.js` | - | 112-307 |
| Runner 类型定义 | `bruno-common/src/runner/types/index.ts` | 1-125 | 1-125 |
| 单请求执行 (桌面端) | `bruno-electron/src/ipc/network/index.js` | 737-1158 | - |
| 单请求执行 (CLI) | `bruno-cli/src/runner/run-single-request.js` | - | - |
| prepareRequest (桌面端) | `bruno-electron/src/ipc/network/prepare-request.js` | - | - |
| prepareRequest (CLI) | `bruno-cli/src/runner/prepare-request.js` | - | - |
| 变量插值 (桌面端) | `bruno-electron/src/ipc/network/interpolate-vars.js` | - | - |
| 变量插值 (CLI) | `bruno-cli/src/runner/interpolate-vars.js` | - | - |

---

## 文档与实现不一致说明

### ❌ 1. 数据表批量参数化（迭代功能）
**文档预期**: 支持 CSV/JSON 数据文件，多轮迭代执行
**实际实现**:
- 仅在类型定义中预留了 `iterationIndex`、`iterationData`
- 两套 Runner 的核心循环均无外层迭代逻辑
- 无数据文件解析逻辑
- 无迭代变量注入逻辑
- 桌面端和 CLI 均无相关 UI/参数

### ❌ 2. 类型定义与实际代码差异
**类型定义**:
```typescript
// 定义了 iterationData 可选字段
iterationData?: any; // todo - csv/json row data
```
**实际执行**:
- Runner 主循环无迭代逻辑
- 单请求执行无 iterationData 注入
- `T_RunnerRequestExecutionResult.iterationIndex` 始终为 0

### ✅ 3. 请求跳转功能（已实现）
**文档预期**: 支持 `bru.setNextRequest()` 跳转
**实际实现**:
- ✅ 两套 Runner 均完整实现跳转功能
- ✅ Pre-request 脚本中可设置
- ✅ Post-response 脚本中可设置
- ✅ Tests 脚本中可设置
- ✅ 支持 `setNextRequest(null)` 终止执行
- ✅ 10000 次跳转防无限循环

### ✅ 4. 变量注入机制（已实现）
**文档预期**: 多层级变量覆盖，脚本中可修改变量
**实际实现**:
- ✅ 7 层变量优先级正确实现
- ✅ Pre-request 修改变量影响当前请求
- ✅ Post-response / Tests 修改变量影响后续请求
- ✅ `bru.setEnvVar()` / `bru.setVar()` API 正常工作

---

## 执行流程完整时序图

```
─────────────────────────────────────────────────────────────────────────
                          Collection Runner 启动
─────────────────────────────────────────────────────────────────────────
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│  🔧 初始化阶段                                                      │
│     ├─ 收集请求列表 (递归/非递归)                                   │
│     ├─ 应用标签过滤                                                │
│     └─ 应用用户自定义排序                                           │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│  🔁 开始请求循环 (WHILE)                                            │
│     桌面端: bruno-electron/src/ipc/network/index.js:1368-1893 │
│     CLI: bruno-cli/src/commands/run.js:680-765                    │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│  ✅ 检查取消信号 → ❌ 终止 (仅桌面端)                              │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│  ⏭️  跳过检查 (gRPC / Prompt 变量) → continue                      │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│  🔴 Pre-request 脚本执行                                            │
│     ├─ ✅ 成功 → 提取 nextRequestName / stopExecution               │
│     ├─ ⏭️  skipRequest → continue                                   │
│     └─ ❌ 错误 → appendScriptErrorResult → throw → catch            │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│  🔵 变量插值 interpolateVars()                                       │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│  📤 发送 HTTP 请求                                                  │
│     ├─ ✅ 2XX 成功 → 解析响应                                       │
│     ├─ ⚠️  4XX/5XX → 有响应体 → 继续执行                            │
│     └─ ❌ 网络错误 → executeRequestOnFailHandler → throw → catch    │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│  🟢 Post-response 脚本执行                                          │
│     ├─ ✅ 成功 → 提取 nextRequestName / stopExecution               │
│     └─ ❌ 错误 → appendScriptErrorResult → ✅ 继续执行               │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│  ✅ 执行 Assertions → 记录结果                                      │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│  🧪 执行 Tests 脚本                                                │
│     ├─ ✅ 成功 → 提取 nextRequestName / stopExecution               │
│     └─ ❌ 错误 → appendScriptErrorResult → ✅ 继续执行               │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│  catch 捕获异常 → 发送 'error' 事件                                 │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│  🛑 stopRunnerExecution 检查 → true → break 终止                    │
│  🛑 --bail 检查 (仅 CLI) → 有失败 → break 终止                        │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│  🎯 跳转决策                                                        │
│     ├─ nextRequestName === null → break 终止                        │
│     ├─ nextRequestName 存在 → 跳转到指定请求                        │
│     └─ 未设置 / 找不到 → index++ 顺序执行                           │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│  循环结束 → 发送 'testrun-ended' 事件 (正常结束)                    │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
─────────────────────────────────────────────────────────────────────────
                          Runner 执行完成
─────────────────────────────────────────────────────────────────────────
```
