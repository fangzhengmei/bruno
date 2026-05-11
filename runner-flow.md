# Bruno Collection Runner 执行流程分析

## 目录
1. [整体架构概览](#整体架构概览)
2. [请求执行顺序](#请求执行顺序)
3. [迭代变量注入机制](#迭代变量注入机制)
4. [失败处理策略](#失败处理策略)
5. [核心代码位置](#核心代码位置)

---

## 整体架构概览

Bruno 的 Collection Runner 采用**主进程 + 渲染进程**的 IPC 通信架构，核心执行逻辑位于 Electron 主进程中。

### 主要组件：
- **触发入口**: `bruno-app/src/components/Sidebar/Collections/Collection/CollectionItem/RunCollectionItem/index.js`
- **Action 层**: `bruno-app/src/providers/ReduxStore/slices/collections/actions.js`
- **核心执行器**: `bruno-electron/src/ipc/network/index.js` (`renderer:run-collection-folder` handler)
- **单请求执行**: `runRequest()` 函数
- **CLI 执行**: `bruno-cli/src/commands/run.js`

---

## 请求执行顺序

### 1. 初始化阶段 (Collection Runner 启动)

**执行流程**：
```
用户点击 Run 按钮
    ↓
runCollectionFolder() action 被调用
    ↓
通过 IPC 调用 renderer:run-collection-folder
    ↓
创建取消令牌 (cancelTokenUid)
    ↓
发送 'testrun-started' 事件到 UI
```

**关键代码** (`bruno-electron/src/ipc/network/index.js:1273-1322`):
```javascript
ipcMain.handle(
    'renderer:run-collection-folder', 
    async (event, folder, collection, environment, runtimeVariables, recursive, delay, tags, selectedRequestUids) => {
        // 1. 初始化配置
        const collectionUid = collection.uid;
        const cancelTokenUid = uuid();
        const brunoConfig = getBrunoConfig(collectionUid, collection);
        const scriptingConfig = get(brunoConfig, 'scripts', {});
        scriptingConfig.runtime = getJsSandboxRuntime(collection);
        const envVars = getEnvVars(environment);
        const processEnvVars = getProcessEnvVars(collectionUid);
        
        // 2. 发送开始事件
        mainWindow.webContents.send('main:run-folder-event', {
            type: 'testrun-started',
            isRecursive: recursive,
            collectionUid,
            folderUid,
            cancelTokenUid
        });
        // ...
    }
);
```

### 2. 请求收集与排序

**排序规则**：
- **递归模式**: 使用 `sortFolder()` + `getAllRequestsInFolderRecursively()` 深度优先
- **非递归模式**: 按 `seq` 属性排序（使用 `sortByNameThenSequence()`）

**过滤机制**：
```javascript
// 标签过滤 (line: 1343-1350)
folderRequests = folderRequests.filter(({ tags: requestTags = [], draft }) => {
    requestTags = draft?.tags || requestTags || [];
    return isRequestTagsIncluded(requestTags, includeTags, excludeTags);
});

// 选中请求过滤 (line: 1352-1366)
if (selectedRequestUids && selectedRequestUids.length > 0) {
    const uidIndexMap = new Map();
    selectedRequestUids.forEach((uid, index) => uidIndexMap.set(uid, index));
    folderRequests = folderRequests
        .filter((request) => uidIndexMap.has(request.uid))
        .sort((a, b) => uidIndexMap.get(a.uid) - uidIndexMap.get(b.uid));
}
```

### 3. 循环执行控制

**核心循环** (`line: 1368-1893`):
```javascript
let currentRequestIndex = 0;
let nJumps = 0; // 防止无限循环的跳转计数器

while (currentRequestIndex < folderRequests.length) {
    // 检查取消信号
    if (abortController.signal.aborted) {
        let error = new Error('Runner execution cancelled');
        error.isCancel = true;
        throw error;
    }
    
    const item = cloneDeep(folderRequests[currentRequestIndex]);
    let nextRequestName;
    
    // 发送 'request-queued' 事件
    // ... 执行单个请求 ...
    
    // 跳转控制
    if (nextRequestName !== undefined) {
        nJumps++;
        if (nJumps > 10000) {
            throw new Error('Too many jumps, possible infinite loop');
        }
        if (nextRequestName === null) {
            break; // 停止执行
        }
        const nextRequestIdx = folderRequests.findIndex((request) => request.name === nextRequestName);
        if (nextRequestIdx >= 0) {
            currentRequestIndex = nextRequestIdx; // 跳转到指定请求
        } else {
            console.error('Could not find request with name \'' + nextRequestName + '\'');
            currentRequestIndex++;
        }
    } else {
        currentRequestIndex++; // 顺序执行
    }
}
```

**跳转控制来源**：
1. **Pre-request script**: `bru.setNextRequest('requestName')`
2. **Post-response script**: `bru.setNextRequest('requestName')`
3. **Tests script**: `bru.setNextRequest('requestName')`
4. **停止执行**: `bru.setNextRequest(null)` 或 `bru.stopRunner()`

---

## 迭代变量注入机制

### 变量层级结构

Runner 采用**多层级变量覆盖**机制，优先级从高到低：

```
Runtime Variables (运行时)
    ↑
Request Variables (请求级)
    ↑
Folder Variables (文件夹级)
    ↑
Environment Variables (环境变量)
    ↑
Collection Variables (集合级)
    ↑
Global Environment Variables (全局环境)
    ↑
Process Environment Variables (系统环境)
```

### 变量注入时机

#### 1. 准备阶段 - `prepareRequest()`
```javascript
// 位置: bruno-electron/src/ipc/network/prepare-request.js
// 作用: 合并 collection/folder/request 级别的变量
```

#### 2. Pre-request 脚本执行后
```javascript
// 位置: bruno-electron/src/ipc/network/index.js:runPreRequest()
// 执行脚本后更新变量:
mainWindow.webContents.send('main:script-environment-update', {
    envVariables: scriptResult.envVariables,
    runtimeVariables: scriptResult.runtimeVariables,
    persistentEnvVariables: scriptResult.persistentEnvVariables,
    requestUid,
    collectionUid
});
```

#### 3. 变量插值 - `interpolateVars()`
```javascript
// 位置: bruno-electron/src/ipc/network/index.js:581
// 在发送 HTTP 请求前执行变量替换
interpolateVars(request, envVars, runtimeVariables, processEnvVars, promptVariables);
```

**插值规则** (`bruno-electron/src/ipc/network/interpolate-vars.js`):
- `{{variableName}}` - 标准变量语法
- 支持嵌套对象访问 `{{user.name}}`
- 支持变量引用其他变量

#### 4. Post-response 脚本执行后
```javascript
// 位置: bruno-electron/src/ipc/network/index.js:runPostResponse()
// 执行 Post-response vars 和 script 后更新变量
```

#### 5. Tests 脚本执行后
```javascript
// 位置: bruno-electron/src/ipc/network/index.js:1091-1107
// 测试脚本执行后更新变量，用于请求间的数据传递
mainWindow.webContents.send('main:script-environment-update', {
    envVariables: testResults.envVariables,
    runtimeVariables: testResults.runtimeVariables,
    requestUid,
    collectionUid
});
```

### 变量操作 API (脚本中使用)

| API | 作用域 | 示例 |
|-----|--------|------|
| `bru.getEnvVar('name')` | 环境变量 | `const token = bru.getEnvVar('accessToken')` |
| `bru.setEnvVar('name', 'value')` | 环境变量 | `bru.setEnvVar('token', res.body.token)` |
| `bru.setVar('name', 'value')` | 运行时变量 | `bru.setVar('userId', 123)` |
| `bru.getVar('name')` | 运行时变量 | `const id = bru.getVar('userId')` |

---

## 失败处理策略

### 1. 请求级别错误处理

#### Pre-request 脚本错误
```javascript
// 位置: bruno-electron/src/ipc/network/index.js:1456-1505
try {
    preRequestScriptResult = await runPreRequest(...);
} catch (error) {
    console.error('Pre-request script error:', error);
    preRequestError = error;
}

// 提取部分执行结果 (tests that passed before error)
if (preRequestError?.partialResults) {
    preRequestScriptResult = preRequestError.partialResults;
}

// 添加错误结果到测试报告
preRequestScriptResult = appendScriptErrorResult('pre-request', preRequestScriptResult, preRequestError);

// 发送错误到 UI
notifyScriptExecution({
    channel: 'main:run-folder-event',
    basePayload: eventData,
    scriptType: 'pre-request',
    error: preRequestError,
    // ...
});

// 脚本错误导致请求失败，抛出异常终止当前请求
if (preRequestError) {
    throw preRequestError;
}
```

#### HTTP 请求错误
```javascript
// 位置: bruno-electron/src/ipc/network/index.js:1650-1694
catch (error) {
    if (axios.isCancel(error)) {
        throw error; // 用户取消请求
    }
    
    if (error?.response) {
        // 服务器返回错误响应 (4XX/5XX) - 继续执行后续流程
        error.response.data = await promisifyStream(error.response.data);
        // ... 处理响应数据 ...
        
        mainWindow.webContents.send('main:run-folder-event', {
            type: 'response-received',
            error: error ? error.message : 'An error occurred while running the request',
            responseReceived: response,
            ...eventData
        });
    } else {
        // 网络错误、DNS 错误等 - 终止当前请求
        await executeRequestOnFailHandler(request, error);
        throw error;
    }
}
```

#### Post-response 脚本错误
```javascript
// 位置: bruno-electron/src/ipc/network/index.js:1697-1754
try {
    postResponseScriptResult = await runPostResponse(...);
} catch (error) {
    console.error('Post-response script error:', error);
    postResponseError = error;
}

// 提取部分执行结果 (tests that passed before error)
if (postResponseError?.partialResults) {
    postResponseScriptResult = postResponseError.partialResults;
}

postResponseScriptResult = appendScriptErrorResult('post-response', postResponseScriptResult, postResponseError);

// 继续执行后续流程 (Assertions, Tests)
```

#### Tests 脚本错误
```javascript
// 位置: bruno-electron/src/ipc/network/index.js:1798-1813
try {
    testResults = await testRuntime.runTests(...);
} catch (error) {
    testError = error;
    if (error.partialResults) {
        testResults = error.partialResults; // 保留已通过的测试结果
    } else {
        testResults = { results: [], ... };
    }
}

testResults = appendScriptErrorResult('test', testResults, testError);
```

### 2. 错误结果追加机制

```javascript
// 位置: bruno-electron/src/ipc/network/index.js:471-502
const appendScriptErrorResult = (scriptType, scriptResult, error) => {
    if (!error) return scriptResult;

    const descriptionMap = {
        'test': 'Test Script Error',
        'post-response': 'Post-Response Script Error',
        'pre-request': 'Pre-Request Script Error'
    };

    const results = [
        ...(scriptResult?.results || []),
        {
            status: 'fail',
            description: descriptionMap[scriptType] || 'Script Error',
            error: error.message || 'An error occurred while executing the script.',
            isScriptError: true // 标记为脚本执行错误 (非断言失败)
        }
    ];

    return { ...(scriptResult || {}), results };
};
```

### 3. Runner 级别的终止条件

#### 主动停止执行
```javascript
// 位置: bruno-electron/src/ipc/network/index.js:1863-1872
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
```

**触发方式**：
- `bru.stopRunner()` - 在任意脚本中调用
- Pre-request / Post-response / Tests 脚本返回 `stopExecution: true`

#### 用户取消执行
```javascript
// 位置: bruno-electron/src/ipc/network/index.js:1372-1376
if (abortController.signal.aborted) {
    let error = new Error('Runner execution cancelled');
    error.isCancel = true;
    throw error;
}
```

#### 无限循环防护
```javascript
// 位置: bruno-electron/src/ipc/network/index.js:1875-1879
nJumps++;
if (nJumps > 10000) {
    throw new Error('Too many jumps, possible infinite loop');
}
```

### 4. 跳过请求机制

```javascript
// 位置: bruno-electron/src/ipc/network/index.js:1515-1530
if (preRequestScriptResult?.skipRequest) {
    mainWindow.webContents.send('main:run-folder-event', {
        type: 'runner-request-skipped',
        error: 'Request has been skipped from pre-request script',
        responseReceived: {
            status: 'skipped',
            statusText: 'request skipped via pre-request script',
            data: null,
            responseTime: 0,
            headers: null
        },
        ...eventData
    });
    currentRequestIndex++;
    continue; // 跳过当前请求，继续下一个
}
```

**触发方式**：`bru.skipRequest()` 在 pre-request 脚本中调用

### 5. 特殊请求跳过

- **gRPC 请求**: Collection Runner 目前不支持 gRPC 请求，会自动跳过
- **含 Prompt 变量的请求**: 需要用户交互输入的请求会被跳过

```javascript
// 位置: bruno-electron/src/ipc/network/index.js:1398-1439
// gRPC 跳过
if (item.type === 'grpc-request') {
    mainWindow.webContents.send('main:run-folder-event', {
        type: 'runner-request-skipped',
        error: 'gRPC requests are skipped in folder/collection runs',
        // ...
    });
    currentRequestIndex++;
    continue;
}

// Prompt 变量跳过
if (promptVars.length > 0) {
    mainWindow.webContents.send('main:run-folder-event', {
        type: 'runner-request-skipped',
        error: 'Request has been skipped due to containing prompt variables',
        // ...
    });
    currentRequestIndex++;
    continue;
}
```

---

## 核心代码位置

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| Runner 主循环 | `bruno-electron/src/ipc/network/index.js` | 1273-1912 |
| 单请求执行 | `bruno-electron/src/ipc/network/index.js` | 737-1158 |
| Pre-request 执行 | `bruno-electron/src/ipc/network/index.js` | 523-635 |
| Post-response 执行 | `bruno-electron/src/ipc/network/index.js` | 637-735 |
| 变量插值 | `bruno-electron/src/ipc/network/interpolate-vars.js` | - |
| CLI Runner | `bruno-cli/src/commands/run.js` | 309-826 |
| Runner UI 入口 | `bruno-app/src/components/Sidebar/Collections/Collection/CollectionItem/RunCollectionItem/index.js` | 1-127 |
| Runner Actions | `bruno-app/src/providers/ReduxStore/slices/collections/actions.js` | 667-720 |

---

## 执行流程总结图

```
Collection Runner 启动
    ↓
┌─────────────────────────────────────────────────────────┐
│  收集并排序请求 (递归/标签/选中过滤)                      │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│  WHILE (还有未执行的请求)                                 │
│    ↓                                                     │
│  ┌───────────────────────────────────────────────────┐  │
│  │  发送 request-queued 事件                          │  │
│  │  检查取消信号 → 终止                                │  │
│  │  跳过 gRPC/Prompt 变量请求                         │  │
│  │  prepareRequest() 合并变量                         │  │
│  └───────────────────────────────────────────────────┘  │
│    ↓                                                     │
│  ┌───────────────────────────────────────────────────┐  │
│  │  🔴 Pre-request 脚本执行                           │  │
│  │    ├─ 提取部分测试结果 (出错前已通过的)            │  │
│  │    ├─ 更新环境变量/运行时变量                      │  │
│  │    ├─ 检查 skipRequest → 跳过当前请求             │  │
│  │    ├─ 检查 stopExecution → 终止 Runner            │  │
│  │    └─ 检查 nextRequestName → 设置跳转目标         │  │
│  └───────────────────────────────────────────────────┘  │
│    ↓                                                     │
│  ┌───────────────────────────────────────────────────┐  │
│  │  🔵 执行变量插值 (interpolateVars)                │  │
│  │  ┌─────────────────────────────────────────────┐  │  │
│  │  │  📤 发送 HTTP 请求                           │  │  │
│  │  │    ├─ 延迟请求 (delay 参数)                 │  │  │
│  │  │    ├─ 网络错误处理                          │  │  │
│  │  │    └─ 4XX/5XX 响应处理                     │  │  │
│  │  └─────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────┘  │
│    ↓                                                     │
│  ┌───────────────────────────────────────────────────┐  │
│  │  🟢 Post-response 脚本执行                         │  │
│  │    ├─ 提取部分测试结果                            │  │
│  │    ├─ 更新环境变量/运行时变量                      │  │
│  │    ├─ 检查 stopExecution → 终止 Runner            │  │
│  │    └─ 检查 nextRequestName → 设置跳转目标         │  │
│  └───────────────────────────────────────────────────┘  │
│    ↓                                                     │
│  ┌───────────────────────────────────────────────────┐  │
│  │  ✅ 执行 Assertions                                │  │
│  │  🧪 执行 Tests 脚本                                │  │
│  │    ├─ 提取部分测试结果                            │  │
│  │    ├─ 更新环境变量/运行时变量                      │  │
│  │    ├─ 检查 stopExecution → 终止 Runner            │  │
│  │    └─ 检查 nextRequestName → 设置跳转目标         │  │
│  └───────────────────────────────────────────────────┘  │
│    ↓                                                     │
│  ┌───────────────────────────────────────────────────┐  │
│  │  确定下一个请求                                     │  │
│  │    ├─ nextRequestName 存在 → 跳转                 │  │
│  │    └─ 不存在 → currentRequestIndex++ (顺序执行)    │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
    ↓
发送 'testrun-ended' 事件
    ↓
Runner 执行完成
```
