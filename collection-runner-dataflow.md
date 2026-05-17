# Bruno Collection Runner 数据驱动流程分析

## 1. 概述

Bruno 的 Collection Runner 是一个用于批量执行 API 请求的核心功能，支持数据驱动测试、变量传递、脚本执行和结果汇总。本文档详细分析其完整执行流程。

---

## 2. 核心入口与调用链

### 2.1 前端触发入口 (React 组件)

**文件位置**: `packages/bruno-app/src/components/RunnerResults/index.jsx:187`

```javascript
const runCollection = () => {
  const savedOrder = get(collection, 'runnerConfiguration.requestItemsOrder', selectedRequestItems);
  dispatch(updateRunnerConfiguration(collection.uid, selectedRequestItems, savedOrder, delay));
  dispatch(runCollectionFolder(collection.uid, null, true, Number(delay), tags, selectedRequestItems));
};
```

### 2.2 Redux Action 层

**文件位置**: `packages/bruno-app/src/providers/ReduxStore/slices/collections/actions.js:667`

```javascript
export const runCollectionFolder = (collectionUid, folderUid, recursive, delay, tags, selectedRequestUids) => (dispatch, getState) => {
  // 1. 获取全局环境变量和集合信息
  const { globalEnvironments, activeGlobalEnvironmentUid } = state.globalEnvironments;
  const collection = findCollectionByUid(state.collections.collections, collectionUid);
  
  // 2. 合并环境变量
  const globalEnvironmentVariables = getGlobalEnvironmentVariables({...});
  collectionCopy.globalEnvironmentVariables = globalEnvironmentVariables;
  
  // 3. 通过 IPC 调用主进程执行
  ipcRenderer.invoke('renderer:run-collection-folder', folder, collectionCopy, environment, runtimeVariables, recursive, delay, tags, selectedRequestUids);
};
```

### 2.3 主进程执行入口

**文件位置**: `packages/bruno-electron/src/ipc/network/index.js:1274`

```javascript
ipcMain.handle('renderer:run-collection-folder', async (event, folder, collection, environment, runtimeVariables, recursive, delay, tags, selectedRequestUids) => {
  // 核心执行逻辑
});
```

---

## 3. 变量装载流程 (Variable Loading)

### 3.1 变量层级与优先级

```
优先级从高到低:
┌─────────────────────────────────────────────────┐
│ 1. Runtime Variables (运行时变量)               │
│    - 通过 bru.setVar() 动态设置                  │
├─────────────────────────────────────────────────┤
│ 2. Request Variables (请求级变量)                │
├─────────────────────────────────────────────────┤
│ 3. Folder Variables (文件夹级变量)               │
├─────────────────────────────────────────────────┤
│ 4. Environment Variables (环境变量)              │
├─────────────────────────────────────────────────┤
│ 5. Collection Variables (集合级变量)             │
├─────────────────────────────────────────────────┤
│ 6. Global Environment Variables (全局环境变量)   │
├─────────────────────────────────────────────────┤
│ 7. Process Environment Variables (进程环境变量)   │
│    - 操作系统环境变量                            │
└─────────────────────────────────────────────────┘
```

### 3.2 变量装载时序

```
┌─────────────────┐     ┌─────────────────────┐     ┌─────────────────────┐
│  Runner 启动    │────▶│  提取环境变量        │────▶│  提取进程环境变量    │
└─────────────────┘     └─────────────────────┘     └─────────────────────┘
                                                             │
                                                             ▼
┌─────────────────┐     ┌─────────────────────┐     ┌─────────────────────┐
│  执行前预处理   │◀────│  初始化运行时变量    │◀────│  提取全局环境变量    │
└─────────────────┘     └─────────────────────┘     └─────────────────────┘
         │
         ▼
┌─────────────────┐     ┌─────────────────────┐     ┌─────────────────────┐
│  Pre-request    │────▶│  变量插值替换        │────▶│  发送实际请求        │
│  Script         │     └─────────────────────┘     └─────────────────────┘
└─────────────────┘
```

### 3.3 变量插值实现

**文件位置**: `packages/bruno-electron/src/ipc/network/interpolate-vars.js`

```javascript
// 核心插值函数 - 在请求发送前替换所有 {{variable}} 占位符
const interpolateVars = (request, envVars, runtimeVariables, processEnvVars, promptVariables) => {
  // 1. URL 插值
  request.url = interpolateString(request.url, interpolationOptions);
  
  // 2. Headers 插值
  Object.keys(request.headers).forEach(key => {
    request.headers[key] = interpolateString(request.headers[key], interpolationOptions);
  });
  
  // 3. Body 插值 (支持多种格式: form-data, x-www-form-urlencoded, raw)
  // 4. Auth 配置插值
  // 5. Query Params 插值
};
```

---

## 4. 迭代执行流程 (Iteration Execution)

### 4.1 请求收集与过滤

**文件位置**: `packages/bruno-electron/src/ipc/network/index.js:1325-1366`

```javascript
// 步骤 1: 递归收集文件夹中的请求
let folderRequests = [];
if (recursive) {
  let sortedFolder = sortFolder(folder);
  folderRequests = getAllRequestsInFolderRecursively(sortedFolder);
} else {
  // 仅当前文件夹
  each(folder.items, (item) => {
    if (item.request && !item.isTransient) {
      folderRequests.push(item);
    }
  });
  folderRequests = sortByNameThenSequence(folderRequests);
}

// 步骤 2: 按 Tag 过滤
if (tags && tags.include && tags.exclude) {
  folderRequests = folderRequests.filter(({ tags: requestTags = [], draft }) => {
    requestTags = draft?.tags || requestTags || [];
    return isRequestTagsIncluded(requestTags, includeTags, excludeTags);
  });
}

// 步骤 3: 按用户选择的 UID 过滤并保持顺序
if (selectedRequestUids && selectedRequestUids.length > 0) {
  const uidIndexMap = new Map();
  selectedRequestUids.forEach((uid, index) => uidIndexMap.set(uid, index));
  
  folderRequests = folderRequests
    .filter((request) => uidIndexMap.has(request.uid))
    .sort((a, b) => uidIndexMap.get(a.uid) - uidIndexMap.get(b.uid));
}
```

### 4.2 主执行循环

**文件位置**: `packages/bruno-electron/src/ipc/network/index.js:1368-1893`

```javascript
let currentRequestIndex = 0;
let nJumps = 0; // 防止无限循环的跳转计数器

while (currentRequestIndex < folderRequests.length) {
  // 检查取消信号
  if (abortController.signal.aborted) {
    throw new Error('Runner execution cancelled');
  }

  const item = cloneDeep(folderRequests[currentRequestIndex]);
  let nextRequestName;
  
  // ========== 1. 预处理阶段 ==========
  mainWindow.webContents.send('main:run-folder-event', {
    type: 'request-queued',
    collectionUid,
    folderUid,
    itemUid: item.uid
  });

  // 跳过 gRPC 请求 (暂不支持)
  if (item.type === 'grpc-request') {
    mainWindow.webContents.send('main:run-folder-event', {
      type: 'runner-request-skipped',
      error: 'gRPC requests are skipped in folder/collection runs',
      ...
    });
    currentRequestIndex++;
    continue;
  }

  // 检查 Prompt 变量 (含 {{$prompt}} 的请求需要人工输入，跳过)
  const promptVars = await extractPromptVariablesForRequest({...});
  if (promptVars.length > 0) {
    mainWindow.webContents.send('main:run-folder-event', {
      type: 'runner-request-skipped',
      error: 'Request has been skipped due to containing prompt variables',
      ...
    });
    currentRequestIndex++;
    continue;
  }

  try {
    // ========== 2. Pre-request Script 执行 ==========
    let preRequestScriptResult;
    try {
      preRequestScriptResult = await runPreRequest(...);
    } catch (error) {
      preRequestError = error;
    }

    // 处理脚本错误 - 保留部分执行结果
    if (preRequestError?.partialResults) {
      preRequestScriptResult = preRequestError.partialResults;
    }
    preRequestScriptResult = appendScriptErrorResult('pre-request', preRequestScriptResult, preRequestError);

    // 发送前置测试结果
    if (preRequestScriptResult?.results) {
      mainWindow.webContents.send('main:run-folder-event', {
        type: 'test-results-pre-request',
        preRequestTestResults: preRequestScriptResult.results,
        ...
      });
    }

    // 脚本控制指令处理
    if (preRequestScriptResult?.nextRequestName !== undefined) {
      nextRequestName = preRequestScriptResult.nextRequestName;  // 跳转指定请求
    }
    if (preRequestScriptResult?.stopExecution) {
      stopRunnerExecution = true;  // 终止整个 Runner
    }
    if (preRequestScriptResult?.skipRequest) {
      // 跳过当前请求
      mainWindow.webContents.send('main:run-folder-event', { type: 'runner-request-skipped', ... });
      currentRequestIndex++;
      continue;
    }

    // ========== 3. 请求延迟 (Delay) ==========
    if (delay && !Number.isNaN(delay) && delay > 0) {
      const delayPromise = new Promise((resolve) => setTimeout(resolve, delay));
      const cancellationPromise = new Promise((_, reject) => {
        abortController.signal.addEventListener('abort', () => reject(new Error('Cancelled')));
      });
      await Promise.race([delayPromise, cancellationPromise]);
    }

    // ========== 4. 发送 HTTP 请求 ==========
    timeStart = Date.now();
    response = await axiosInstance(request);
    timeEnd = Date.now();

    // 解析响应数据
    const { data, dataBuffer } = parseDataFromResponse(response, request.__brunoDisableParsingResponseJson);
    response.data = data;
    response.dataBuffer = dataBuffer;

    // 发送响应结果事件
    mainWindow.webContents.send('main:run-folder-event', {
      type: 'response-received',
      responseReceived: {
        status: response.status,
        statusText: response.statusText,
        headers: response.headers,
        duration: timeEnd - timeStart,
        data: response.data,
        ...
      },
      ...
    });

    // ========== 5. Post-response Script 执行 ==========
    let postResponseScriptResult;
    try {
      postResponseScriptResult = await runPostResponse(...);
    } catch (error) {
      postResponseError = error;
    }

    // 处理脚本跳转指令
    if (postResponseScriptResult?.nextRequestName !== undefined) {
      nextRequestName = postResponseScriptResult.nextRequestName;
    }
    if (postResponseScriptResult?.stopExecution) {
      stopRunnerExecution = true;
    }

    // 发送后置测试结果
    if (postResponseScriptResult?.results) {
      mainWindow.webContents.send('main:run-folder-event', {
        type: 'test-results-post-response',
        postResponseTestResults: postResponseScriptResult.results,
        ...
      });
    }

    // ========== 6. Assertions (断言) 执行 ==========
    const assertions = get(item, 'request.assertions');
    if (assertions) {
      const assertRuntime = new AssertRuntime({ runtime: scriptingConfig?.runtime });
      const results = assertRuntime.runAssertions(assertions, request, response, envVars, runtimeVariables, processEnvVars);
      mainWindow.webContents.send('main:run-folder-event', {
        type: 'assertion-results',
        assertionResults: results,
        ...
      });
    }

    // ========== 7. Tests (测试脚本) 执行 ==========
    const testFile = get(request, 'tests');
    if (typeof testFile === 'string') {
      const testRuntime = new TestRuntime({ runtime: scriptingConfig?.runtime });
      testResults = await testRuntime.runTests(...);
      
      if (testResults?.nextRequestName !== undefined) {
        nextRequestName = testResults.nextRequestName;
      }
      
      mainWindow.webContents.send('main:run-folder-event', {
        type: 'test-results',
        testResults: testResults.results,
        ...
      });
    }

  } catch (error) {
    // 错误处理
    mainWindow.webContents.send('main:run-folder-event', {
      type: 'error',
      error: error ? error.message : 'An error occurred while running the request',
      ...
    });
  }

  // ========== 8. 终止检查 (Stop Execution) ==========
  if (stopRunnerExecution) {
    deleteCancelToken(cancelTokenUid);
    mainWindow.webContents.send('main:run-folder-event', {
      type: 'testrun-ended',
      collectionUid,
      folderUid,
      statusText: 'collection run was terminated!',
      runCompletionTime: new Date().toISOString()
    });
    break;
  }

  // ========== 9. 跳转逻辑 (Jump Logic) ==========
  if (nextRequestName !== undefined) {
    nJumps++;
    if (nJumps > 10000) {  // 防止无限循环
      throw new Error('Too many jumps, possible infinite loop');
    }
    if (nextRequestName === null) {
      break;  // 终止执行
    }
    // 查找目标请求并跳转
    const nextRequestIdx = folderRequests.findIndex((request) => request.name === nextRequestName);
    if (nextRequestIdx >= 0) {
      currentRequestIndex = nextRequestIdx;
    } else {
      console.error('Could not find request with name \'' + nextRequestName + '\'');
      currentRequestIndex++;
    }
  } else {
    currentRequestIndex++;  // 正常顺序执行
  }
}
```

---

## 5. 脚本控制指令 (Script Control)

### 5.1 可用指令列表

| 指令 | 作用域 | 说明 |
|------|--------|------|
| `bru.setVar(name, value)` | Pre/Post | 设置运行时变量 |
| `bru.setEnvVar(name, value)` | Pre/Post | 设置环境变量 |
| `bru.skipRequest()` | Pre-request | 跳过当前请求 |
| `bru.stopExecution()` | Pre/Post | 终止整个 Runner 执行 |
| `bru.runNextRequest(requestName)` | Pre/Post | 跳转到指定名称的请求 |
| `bru.runNextRequest(null)` | Pre/Post | 终止执行 |

### 5.2 跳转流程示例

```
正常顺序执行:
[Req A] ──▶ [Req B] ──▶ [Req C] ──▶ [结束]

使用 bru.runNextRequest('Req C') 跳转:
[Req A] ──▶ [Req B] ──┐
                       │
        ┌──────────────┘
        ▼
      [Req C] ──▶ [结束]

使用 bru.stopExecution() 终止:
[Req A] ──▶ [Req B] ──▶ [终止]
```

---

## 6. 结果汇总流程 (Result Aggregation)

### 6.1 事件流与数据结构

```
事件类型:
┌─────────────────────────┐
│ testrun-started         │  开始执行
├─────────────────────────┤
│ request-queued          │  请求入队
├─────────────────────────┤
│ test-results-pre-request│  前置脚本测试结果
├─────────────────────────┤
│ request-sent            │  请求已发送
├─────────────────────────┤
│ response-received       │  响应已接收
├─────────────────────────┤
│ test-results-post-response│ 后置脚本测试结果
├─────────────────────────┤
│ assertion-results       │  断言结果
├─────────────────────────┤
│ test-results            │  测试脚本结果
├─────────────────────────┤
│ error                   │  执行错误
├─────────────────────────┤
│ runner-request-skipped  │  请求被跳过
├─────────────────────────┤
│ testrun-ended           │  执行结束
└─────────────────────────┘
```

### 6.2 前端结果聚合逻辑

**文件位置**: `packages/bruno-common/src/runner/runner-summary.ts`

```typescript
export const getRunnerSummary = (results: T_RunnerRequestExecutionResult[]): T_RunSummary => {
  let totalRequests = 0;
  let passedRequests = 0;
  let failedRequests = 0;
  let errorRequests = 0;
  let skippedRequests = 0;
  let totalAssertions = 0;
  let passedAssertions = 0;
  let failedAssertions = 0;
  let totalTests = 0;
  let passedTests = 0;
  let failedTests = 0;
  // ... 更多统计项

  for (const result of results || []) {
    const { status, testResults, assertionResults, ... } = result;
    totalRequests += 1;
    totalTests += Number(testResults?.filter((r) => !r.isScriptError).length) || 0;
    totalAssertions += Number(assertionResults?.length) || 0;

    if (status === 'skipped') {
      skippedRequests += 1;
      continue;
    }

    // 检查是否有任何失败
    let anyFailed = false;
    
    // 统计测试结果
    for (const testResult of testResults || []) {
      if (testResult.isScriptError) {
        anyFailed = true;
        continue;
      }
      if (testResult.status === 'pass') {
        passedTests += 1;
      } else {
        anyFailed = true;
        failedTests += 1;
      }
    }

    // 统计断言结果
    for (const assertionResult of assertionResults || []) {
      if (assertionResult.status === 'pass') {
        passedAssertions += 1;
      } else {
        anyFailed = true;
        failedAssertions += 1;
      }
    }

    // 统计 Pre-request 测试结果
    for (const preRequestTestResult of preRequestTestResults || []) {
      if (preRequestTestResult.isScriptError) {
        anyFailed = true;
        continue;
      }
      if (preRequestTestResult.status === 'pass') {
        passedPreRequestTests += 1;
      } else {
        anyFailed = true;
        failedPreRequestTests += 1;
      }
    }

    // 统计 Post-response 测试结果
    // ... 类似逻辑

    // 最终判定请求状态
    if (!anyFailed && status !== 'error') {
      passedRequests += 1;
    } else if (anyFailed) {
      failedRequests += 1;
    } else {
      errorRequests += 1;
    }
  }

  return {
    totalRequests, passedRequests, failedRequests, errorRequests, skippedRequests,
    totalAssertions, passedAssertions, failedAssertions,
    totalTests, passedTests, failedTests,
    // ... 更多统计
  };
};
```

### 6.3 结果类型定义

**文件位置**: `packages/bruno-common/src/runner/types/index.ts`

```typescript
export type T_RunnerRequestExecutionResult = {
  iterationIndex: number;
  name: string;
  path: string;
  request: T_Request;
  response: T_Response;
  status: null | undefined | string;  // 'pass' | 'fail' | 'error' | 'skipped'
  error: null | undefined | string;
  assertionResults?: T_AssertionResult[];
  testResults?: T_TestResult[];
  preRequestTestResults?: T_TestResult[];
  postResponseTestResults?: T_TestResult[];
  runDuration: number;
};

export type T_RunSummary = {
  totalRequests: number;
  passedRequests: number;
  failedRequests: number;
  errorRequests: number;
  skippedRequests: number;
  totalAssertions: number;
  passedAssertions: number;
  failedAssertions: number;
  totalTests: number;
  passedTests: number;
  failedTests: number;
  // ... pre-request 和 post-response 统计
};
```

---

## 7. 失败中断逻辑 (Failure Interruption)

### 7.1 终止触发场景

| 场景 | 触发位置 | 说明 |
|------|----------|------|
| 用户手动取消 | 主循环开始 | 点击 "Cancel Execution" 按钮 |
| `bru.stopExecution()` | Pre-request 后 | 脚本主动调用终止 |
| `bru.stopExecution()` | Post-response 后 | 脚本主动调用终止 |
| 跳转次数超限 | 跳转逻辑 | `nJumps > 10000` 防止无限循环 |

### 7.2 取消令牌机制

```javascript
// 初始化取消令牌
const cancelTokenUid = uuid();
const abortController = new AbortController();
saveCancelToken(cancelTokenUid, abortController);

// 监听取消信号
abortController.signal.addEventListener('abort', () => {
  if (currentAbortController) {
    currentAbortController.abort();  // 同时取消当前请求
  }
});

// 主循环每次迭代检查
while (currentRequestIndex < folderRequests.length) {
  if (abortController.signal.aborted) {
    let error = new Error('Runner execution cancelled');
    error.isCancel = true;
    throw error;
  }
  // ... 执行请求
}

// Delay 期间也可取消
if (delay && delay > 0) {
  const delayPromise = new Promise((resolve) => setTimeout(resolve, delay));
  const cancellationPromise = new Promise((_, reject) => {
    abortController.signal.addEventListener('abort', () => reject(new Error('Cancelled')));
  });
  await Promise.race([delayPromise, cancellationPromise]);  // 任一完成则继续
}
```

### 7.3 终止清理流程

```
┌─────────────────┐
│  收到终止信号   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐     ┌─────────────────────┐     ┌─────────────────────┐
│  取消当前 HTTP  │────▶│  删除 Cancel Token   │────▶│  发送 testrun-ended │
│  请求 (Abort)   │     └─────────────────────┘     │  事件到前端         │
└─────────────────┘                                 └─────────────────────┘
                                                             │
                                                             ▼
                                                    ┌─────────────────────┐
                                                    │  前端更新 UI 状态   │
                                                    │  - 显示终止原因     │
                                                    │  - 显示已执行结果   │
                                                    └─────────────────────┘
```

---

## 8. 完整执行流程图

```
┌───────────────────────────────────────────────────────────────────────────────────────┐
│                             COLLECTION RUNNER 完整流程                                  │
└───────────────────────────────────────────────────────────────────────────────────────┘

  初始化阶段
    ┌─────────┐     ┌─────────────┐     ┌───────────────┐     ┌───────────────┐
    │  触发   │────▶│  装载变量   │────▶│  收集请求     │────▶│  过滤排序     │
    │  Run    │     │  (7层)      │     │  (递归/平级)  │     │  (Tag/UID)    │
    └─────────┘     └─────────────┘     └───────────────┘     └───────────────┘
                                                                      │
                                                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────────┐
  │                                   WHILE 主循环                                      │
  │  ┌─────────────────────────────────────────────────────────────────────────────┐  │
  │  │  0. 检查取消信号 ──(取消)──▶ 抛出异常，终止执行                               │  │
  │  └─────────────────────────────────────────────────────────────────────────────┘  │
  │                                      │                                            │
  │                                      ▼                                            │
  │  ┌─────────────────────────────────────────────────────────────────────────────┐  │
  │  │  1. 预处理检查                                                               │  │
  │  │     - 跳过 gRPC 请求                                                         │  │
  │  │     - 跳过含 Prompt 变量的请求                                               │  │
  │  └─────────────────────────────────────────────────────────────────────────────┘  │
  │                                      │                                            │
  │                                      ▼                                            │
  │  ┌─────────────────────────────────────────────────────────────────────────────┐  │
  │  │  2. Pre-request Script 执行                                                 │  │
  │  │     ├─> 执行脚本                                                             │  │
  │  │     ├─> 更新变量 (setVar/setEnvVar)                                          │  │
  │  │     ├─> 检查控制指令:                                                        │  │
  │  │     │    bru.skipRequest()  ──▶ 跳过当前请求，continue                       │  │
  │  │     │    bru.stopExecution() ──▶ 设置 stopRunnerExecution = true             │  │
  │  │     │    bru.runNextRequest()  ──▶ 设置 nextRequestName                      │  │
  │  │     └─> 发送前置测试结果事件                                                 │  │
  │  └─────────────────────────────────────────────────────────────────────────────┘  │
  │                                      │                                            │
  │                                      ▼                                            │
  │  ┌─────────────────────────────────────────────────────────────────────────────┐  │
  │  │  3. 变量插值替换                                                             │  │
  │  │     - URL、Headers、Body、Auth、QueryParams                                   │  │
  │  └─────────────────────────────────────────────────────────────────────────────┘  │
  │                                      │                                            │
  │                                      ▼                                            │
  │  ┌─────────────────────────────────────────────────────────────────────────────┐  │
  │  │  4. 请求延迟 (Delay)                                                         │  │
  │  │     - 支持取消延迟                                                            │  │
  │  └─────────────────────────────────────────────────────────────────────────────┘  │
  │                                      │                                            │
  │                                      ▼                                            │
  │  ┌─────────────────────────────────────────────────────────────────────────────┐  │
  │  │  5. 发送 HTTP 请求                                                           │  │
  │  │     ├─> 记录开始时间                                                         │  │
  │  │     ├─> axios 发送请求                                                       │  │
  │  │     ├─> 记录结束时间                                                         │  │
  │  │     └─> 发送 response-received 事件                                          │  │
  │  └─────────────────────────────────────────────────────────────────────────────┘  │
  │                                      │                                            │
  │                                      ▼                                            │
  │  ┌─────────────────────────────────────────────────────────────────────────────┐  │
  │  │  6. Post-response Script 执行                                                │  │
  │  │     ├─> 执行脚本                                                             │  │
  │  │     ├─> 更新变量                                                             │  │
  │  │     ├─> 检查控制指令: stopExecution / runNextRequest                         │  │
  │  │     └─> 发送后置测试结果事件                                                 │  │
  │  └─────────────────────────────────────────────────────────────────────────────┘  │
  │                                      │                                            │
  │                                      ▼                                            │
  │  ┌─────────────────────────────────────────────────────────────────────────────┐  │
  │  │  7. Assertions 执行                                                          │  │
  │  │     - 运行所有断言，收集 pass/fail 结果                                       │  │
  │  └─────────────────────────────────────────────────────────────────────────────┘  │
  │                                      │                                            │
  │                                      ▼                                            │
  │  ┌─────────────────────────────────────────────────────────────────────────────┐  │
  │  │  8. Tests 脚本执行                                                           │  │
  │  │     ├─> 运行测试脚本                                                         │  │
  │  │     ├─> 检查控制指令: runNextRequest                                         │  │
  │  │     └─> 发送测试结果事件                                                     │  │
  │  └─────────────────────────────────────────────────────────────────────────────┘  │
  │                                      │                                            │
  │                                      ▼                                            │
  │  ┌─────────────────────────────────────────────────────────────────────────────┐  │
  │  │  9. 终止检查                                                                 │  │
  │  │     stopRunnerExecution == true  ──▶ 发送 testrun-ended，break 循环           │  │
  │  └─────────────────────────────────────────────────────────────────────────────┘  │
  │                                      │                                            │
  │                                      ▼                                            │
  │  ┌─────────────────────────────────────────────────────────────────────────────┐  │
  │  │  10. 跳转逻辑                                                                │  │
  │  │      ├─> nextRequestName 已设置                                              │  │
  │  │      │    ├─> nJumps++，超过 10000 抛出异常                                  │  │
  │  │      │    ├─> nextRequestName == null ──▶ break 终止                         │  │
  │  │      │    └─> 查找目标请求 ──▶ currentRequestIndex = 目标索引                 │  │
  │  │      └─> 未设置 ──▶ currentRequestIndex++ 顺序执行                           │  │
  │  └─────────────────────────────────────────────────────────────────────────────┘  │
  └───────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  收尾阶段
    ┌─────────────────┐     ┌─────────────────────┐     ┌─────────────────────┐
    │  删除 Cancel   │────▶│  发送 testrun-ended  │────▶│  前端汇总统计       │
    │  Token         │     │  事件                │     │  (getRunnerSummary) │
    └─────────────────┘     └─────────────────────┘     └─────────────────────┘
```

---

## 9. 关键文件索引

| 功能模块 | 文件路径 |
|---------|---------|
| Runner 前端 UI | `packages/bruno-app/src/components/RunnerResults/index.jsx` |
| Runner Redux Actions | `packages/bruno-app/src/providers/ReduxStore/slices/collections/actions.js` |
| Runner 主进程核心逻辑 | `packages/bruno-electron/src/ipc/network/index.js:1274` |
| 变量插值 | `packages/bruno-electron/src/ipc/network/interpolate-vars.js` |
| 运行时类型定义 | `packages/bruno-common/src/runner/types/index.ts` |
| 结果汇总逻辑 | `packages/bruno-common/src/runner/runner-summary.ts` |
| Pre-request Script 执行 | `packages/bruno-electron/src/ipc/network/index.js:523` |
| Post-response Script 执行 | `packages/bruno-electron/src/ipc/network/index.js:637` |
| 单个请求执行 | `packages/bruno-electron/src/ipc/network/index.js:737` |

---

## 10. 总结

Bruno Collection Runner 的设计特点：

1. **多层级变量系统**: 7层变量优先级，支持灵活的数据驱动测试
2. **脚本控制能力**: 通过 `bru.*` API 实现跳转、跳过、终止等流程控制
3. **完整的结果体系**: 区分 pre-request、post-response、assertions、tests 四类测试结果
4. **安全保护机制**: 10000次跳转限制防止无限循环，AbortController 支持随时取消
5. **事件驱动架构**: 通过 IPC 事件实时推送执行状态，前端响应式更新 UI
