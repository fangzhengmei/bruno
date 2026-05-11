# Bruno 脚本沙箱机制分析报告

## 事实修正

### 修正 1：自定义断言的响应对象真实实现

**原描述**：使用 `BrunoResponse` 类
**真实实现**：`createResponseParser()` 函数创建的函数式对象

```javascript
// packages/bruno-js/src/utils.js:110-128
const createResponseParser = (response = {}) => {
  // res 本身是一个 lodash get 风格的取值函数
  const res = (expr, ...fns) => {
    return get(response.data, expr, ...fns);
  };

  // 挂载属性供断言使用
  res.status = response.status;
  res.statusText = response.statusText;
  res.headers = response.headers;
  res.body = response.data;
  res.responseTime = response.responseTime;
  res.url = response.request ? response.request.protocol + '//' + response.request.host + response.request.path : null;

  // 额外提供 jq 风格查询方法
  res.jq = (expr) => {
    const output = jsonQuery(expr, { data: response.data });
    return output ? output.value : null;
  };

  return res;
};
```

**关键差异**：
- 不是 `BrunoResponse` 类实例，而是普通函数挂载属性
- 函数本身可直接用于 `res('data.items[0].name')` 风格的路径取值
- 提供 `.jq(expr)` 方法支持 jsonQuery 语法

### 修正 2：Pre-request 脚本报错后的短路行为

**执行位置**：`packages/bruno-electron/src/ipc/network/index.js:798-844`

```javascript
let preRequestScriptResult = null;
let preRequestError = null;
try {
  preRequestScriptResult = await runPreRequest(...);
} catch (error) {
  preRequestError = error;
}

// 收集部分测试结果（即使脚本出错前已执行的测试）
if (preRequestError?.partialResults) {
  preRequestScriptResult = preRequestError.partialResults;
}

// 发送测试结果到前端
preRequestScriptResult = appendScriptErrorResult('pre-request', preRequestScriptResult, preRequestError);
if (preRequestScriptResult?.results) {
  mainWindow.webContents.send('main:run-request-event', {
    type: 'test-results-pre-request',
    results: preRequestScriptResult.results,
    ...
  });
}

// 【关键短路】pre-request 出错后直接 reject，终止后续流程
if (preRequestError) {
  return Promise.reject(preRequestError);
}

// 后续流程（永远不会执行）：
// configureRequest() → 构建 axios 实例
// 发送 HTTP 请求
// post-response 脚本
// 断言执行
// 测试脚本
```

**短路行为要点**：
1. **发送测试结果**：即使脚本出错，出错前执行的 `test()` 结果仍会发送到前端展示
2. **立即终止**：`return Promise.reject()` 直接跳出整个请求流程
3. **无 HTTP 请求**：不会发送实际的 HTTP 请求
4. **影响范围**：后续的变量插值、请求发送、post-response、断言、测试全部跳过

### 修正 3：Post-response 后 Assertions 与 Tests 的执行顺序

**执行位置**：
- 桌面端：`packages/bruno-electron/src/ipc/network/index.js:1025-1079`
- CLI 端：`packages/bruno-cli/src/runner/run-single-request.js:818-860`

**真实执行顺序**：
```
收到响应 → 解析响应数据
    ↓
1. 【先】AssertRuntime.runAssertions() → 可视化断言
    ├─ 遍历 assertion 数组
    ├─ 在沙箱中求值表达式
    ├─ 使用 chai 断言验证
    └─ 收集断言结果
    ↓
2. 【后】TestRuntime.runTests() → 测试脚本
    ├─ 创建 Bru/BrunoRequest/BrunoResponse 上下文
    ├─ 注入 chai 断言库
    ├─ 在沙箱中执行测试脚本
    └─ 收集 test() 函数的执行结果
```

**代码实证**（bruno-electron）：
```javascript
// 第 1 步：先执行可视化断言
const assertions = get(request, 'assertions');
if (assertions) {
  const assertRuntime = new AssertRuntime({ runtime: scriptingConfig?.runtime });
  const results = assertRuntime.runAssertions(assertions, request, response, ...);
  // 发送断言结果
}

// 第 2 步：后执行测试脚本
const testFile = get(request, 'tests');
if (typeof testFile === 'string') {
  const testRuntime = new TestRuntime({ runtime: scriptingConfig?.runtime });
  try {
    testResults = await testRuntime.runTests(testFile, request, response, ...);
  } catch (error) {
    // 错误处理
  }
}
```

**执行顺序结论**：**Assertions 先于 Tests 执行**

### 修正 4：Assertions 变量可见性的完整证据链

**原结论**：Assertions 不包含 post-response 中 `bru.setVar()` 设置的变量
**真实结论**：**Assertions 完全可以读取到 post-response 中通过 `bru.setVar()` 写入的 runtime 变量**

#### 最小证据链

**证据点 1：对象引用传递（关键）**
```javascript
// packages/bruno-electron/src/ipc/network/index.js:980-1034
// 调用 runPostResponse 时，runtimeVariables 以【对象引用】形式传入
postResponseScriptResult = await runPostResponse(request,
  response,
  requestUid,
  envVars,           // 对象引用
  collectionPath,
  collection,
  collectionUid,
  runtimeVariables,  // ← 关键：对象引用传递
  processEnvVars,
  scriptingConfig,
  runRequestByItemPathname);

// 后续调用 runAssertions 时，传递【同一个】runtimeVariables 对象引用
const results = assertRuntime.runAssertions(assertions,
  request,
  response,
  envVars,
  runtimeVariables,  // ← 同一个对象引用，包含 post-response 中写入的值
  processEnvVars);
```

**证据点 2：Bru.setVar() 直接修改引用对象**
```javascript
// packages/bruno-js/src/bru.js:283-296
setVar(key, value) {
  // ... 参数校验
  this.runtimeVariables[key] = value;  // ← 直接修改传入的引用对象
}

// Bru 构造函数直接保存引用，不做拷贝
constructor({
  runtimeVariables,  // 外部传入的引用
  ...
}) {
  this.runtimeVariables = runtimeVariables;  // ← 直接赋值引用，不创建副本
  // ...
}
```

**证据点 3：ScriptRuntime 返回修改后的引用**
```javascript
// packages/bruno-js/src/runtime/script-runtime.js:90-101
const buildRequestScriptResult = () => ({
  request,
  envVariables: cleanJson(envVariables),         // 只是序列化用于展示
  runtimeVariables: cleanJson(runtimeVariables), // 只是序列化用于展示
  // 注意：返回值只是用于结果展示，原始引用已经在脚本执行时被修改
});
```

**证据点 4：AssertRuntime 直接展开 runtimeVariables 到上下文**
```javascript
// packages/bruno-js/src/runtime/assert-runtime.js:443-453
const context = {
  ...globalEnvironmentVariables,
  ...collectionVariables,
  ...envVariables,
  ...folderVariables,
  ...requestVariables,
  ...oauth2CredentialVariables,
  ...runtimeVariables,  // ← 关键：直接展开已被修改的 runtimeVariables 对象
  ...processEnvVars,
  ...bruContext
};
```

#### 完整变量传递链路

```
┌─ 主流程调用 runPostResponse(runtimeVariables)
│  └─ ScriptRuntime.runResponseScript(script, ..., runtimeVariables, ...)
│     └─ new Bru({ ..., runtimeVariables, ... })  ← 引用传递，不拷贝
│        └─ 用户脚本执行：bru.setVar('myKey', 'myValue')
│           └─ this.runtimeVariables['myKey'] = 'myValue'  ← 修改原始引用对象
│
└─ 主流程继续调用 runAssertions(..., runtimeVariables, ...)
   └─ AssertRuntime.runAssertions(..., runtimeVariables, ...)
      └─ new Bru({ ..., runtimeVariables, ... })  ← 同一个引用，包含 'myKey'
         └─ context = { ..., ...runtimeVariables }  ← 展开到上下文
            └─ evaluateJsExpressionBasedOnRuntime(expr, context)
               └─ 表达式中可以直接使用 myKey 变量
```

**关键设计结论**：JavaScript 按对象引用传递的特性是变量可见的根本原因。没有任何中间步骤做对象深拷贝，post-response 脚本中对 `runtimeVariables` 的修改是**原地修改**，对后续所有环节（Assertions、Tests）天然可见。

---

## 执行链路

### 完整请求生命周期链路

```
用户点击发送请求
    ↓
1. Pre-request 脚本执行
   ├─ ScriptRuntime.runRequestScript()
   ├─ 上下文：Bru + BrunoRequest
   ├─ 支持 test() 断言
   └─ ✘ 出错 → 短路 reject，后续全部跳过
    ↓
2. 变量插值
   └─ interpolateVars() → 替换 {{var}} 占位符
    ↓
3. HTTP 请求发送
   └─ axiosInstance(request)
    ↓
4. 收到响应
   ├─ 解析响应数据
   └─ 保存 Cookies（如启用）
    ↓
5. Post-response 脚本执行
   ├─ ScriptRuntime.runResponseScript()
   ├─ 上下文：Bru + BrunoRequest + BrunoResponse
   ├─ 支持 test() 断言
   ├─ ✘ 出错 → 记录错误但继续后续流程
   └─ ✅ bru.setVar() → 原地修改 runtimeVariables 引用
    ↓
6. Assertions 可视化断言执行【先】
   ├─ AssertRuntime.runAssertions()
   ├─ 上下文：Bru + BrunoRequest + createResponseParser() 返回的 res 对象
   ├─ ✅ runtimeVariables 已包含 post-response 中设置的变量
   ├─ 遍历所有启用的断言
   └─ 收集 pass/fail 结果
    ↓
7. Tests 测试脚本执行【后】
   ├─ TestRuntime.runTests()
   ├─ 上下文：Bru + BrunoRequest + BrunoResponse
   ├─ 完整 chai 支持（expect/assert）
   ├─ 支持 jwt 库
   └─ 收集 test() 函数执行结果
    ↓
8. 返回最终结果
   ├─ 响应数据
   ├─ 断言结果
   ├─ 测试结果
   └─ 变量更新
```

### 关键执行分支

**分支 1：Pre-request 脚本出错**
```
runPreRequest() 抛出异常
    ↓
捕获异常 → 保存 preRequestError
    ↓
提取 partialResults（出错前的测试结果）
    ↓
发送 test-results-pre-request 事件
    ↓
发送脚本错误通知
    ↓
✅ return Promise.reject(preRequestError) → 短路终止
    ↓
后续流程全部跳过（无 HTTP 请求、无 post-response、无 assertions、无 tests）
```

**分支 2：Post-response 脚本出错**
```
runPostResponse() 抛出异常
    ↓
捕获异常 → 保存 postResponseError
    ↓
提取 partialResults（出错前的测试结果）
    ↓
发送 test-results-post-response 事件
    ↓
❌ 不短路 → 继续执行
    ↓
继续执行 Assertions → 继续执行 Tests → 返回完整结果
```

**分支 3：Tests 脚本出错**
```
runTests() 抛出异常
    ↓
捕获异常 → 保存 testError
    ↓
提取 partialResults（出错前的测试结果）
    ↓
✅ 不影响断言结果（Assertions 已执行完）
    ↓
返回：断言结果 + 部分测试结果 + 错误信息
```

---

## 影响说明

### 影响 1：Pre-request 错误对测试结果的影响

| 场景 | HTTP 请求 | Post-response | Assertions | Tests | 已执行的 test() 结果 |
|------|-----------|---------------|------------|-------|---------------------|
| Pre-request 正常 | ✅ 发送 | ✅ 执行 | ✅ 执行 | ✅ 执行 | 全部展示 |
| Pre-request 出错 | ❌ 不发送 | ❌ 不执行 | ❌ 不执行 | ❌ 不执行 | ✅ 出错前的结果会展示 |

**说明**：即使 pre-request 脚本中途出错，出错前已经执行的 `test()` 函数结果会被收集并展示在前端，不会因为脚本错误而丢失。

### 影响 2：执行顺序对变量的影响

```
时序：Post-response → Assertions → Tests

变量可见性：
├─ Post-response 中 bru.setVar(key, value) 设置的变量
│   ├─ ✅ Assertions 可见（对象引用传递，原地修改）
│   └─ ✅ Tests 可见
└─ Assertions 中只能使用内置变量，不能设置变量
    └─ Tests 中设置的变量不会反向影响 Assertions（已执行完）
```

**关键澄清**：
- ✅ **Assertions 完全可以读取** post-response 脚本中通过 `bru.setVar()` 设置的 runtime 变量
- ❌ 原结论"不包含"是错误的，已通过完整代码证据链修正
- 机制本质：JavaScript 对象按引用传递 + Bru 类直接修改引用不创建副本

### 影响 3：断言结果的独立性

| 错误发生位置 | Assertions 结果 | Tests 结果 |
|-------------|----------------|-----------|
| Pre-request 出错 | ❌ 全部不执行 | ❌ 全部不执行 |
| Post-response 出错 | ✅ 正常执行 | ✅ 正常执行 |
| Assertions 内部出错 | ✅ 逐个断言捕获异常，单个失败不影响其他 | ✅ 正常执行 |
| Tests 脚本出错 | ✅ 已执行完，不受影响 | ✅ 出错前的结果会展示 |

**关键设计**：
1. Assertions 执行时每个断言都有独立的 try-catch，单个断言失败不会影响其他断言
2. Tests 执行失败不会回溯影响 Assertions 结果（顺序决定）
3. Post-response 失败不影响 Assertions 和 Tests 执行

### 影响 4：响应对象的 API 差异

| 场景 | 使用对象 | API 能力 |
|------|---------|---------|
| Post-response 脚本 | BrunoResponse 类实例 | 完整的响应操作 API |
| Tests 测试脚本 | BrunoResponse 类实例 | 完整的响应操作 API |
| Assertions 可视化断言 | createResponseParser() 返回的函数 | 仅支持：<br>- `res(expr)` 路径取值<br>- `res.jq(expr)` jq 查询<br>- 直接访问属性：status/headers/body 等 |

**断言表达式示例**：
```javascript
// 支持的表达式写法
res.status === 200
res('data.items').length > 0
res.jq('.data.items[0].name') === 'expected'
res.headers['content-type'].includes('json')

// 同时支持 post-response 中设置的变量
// 前提：post-response 脚本执行了 bru.setVar('expectedCount', 10)
res('data.items').length === expectedCount
```

---

## 附录：原始章节（供参考）

### 一、脚本注入时机（原始）

Bruno 支持三种脚本执行时机，分别在请求生命周期的不同阶段注入执行：

#### 1.1 请求前脚本（Pre-request Script）

**注入时机**：请求发送前执行，用于修改请求参数、设置变量等。

**调用入口**：`ScriptRuntime.runRequestScript()` (`packages/bruno-js/src/runtime/script-runtime.js:18`)

**执行流程**：
1. 构建 Bru 上下文对象，包含所有层级的变量
2. 构建 BrunoRequest 对象，暴露请求属性和方法
3. 注入 chai 断言库和 test 结果收集方法
4. 根据 runtime 配置选择沙箱执行
5. 执行完成后返回更新后的变量和测试结果

#### 1.2 请求后脚本（Post-response Script）

**注入时机**：收到响应后执行，用于处理响应数据、断言验证等。

**调用入口**：`ScriptRuntime.runResponseScript()` (`packages/bruno-js/src/runtime/script-runtime.js:151`)

**执行流程**：
1. 构建 Bru 上下文对象（同请求前）
2. 构建 BrunoRequest 和 BrunoResponse 对象
3. 注入 chai 断言库和 test 结果收集方法
4. 根据 runtime 配置选择沙箱执行
5. 执行完成后返回更新后的变量和测试结果

#### 1.3 自定义断言（Custom Assertions）

**注入时机**：响应返回后，与请求后脚本独立执行，用于可视化断言配置。

**调用入口**：`AssertRuntime.runAssertions()` (`packages/bruno-js/src/runtime/assert-runtime.js:408`)

### 二、运行沙箱（原始）

#### 2.1 QuickJS 沙箱（默认）

**核心实现**：`executeQuickJsVmAsync()` (`packages/bruno-js/src/sandbox/quickjs/index.js:92`)

**特性**：
- 基于 WebAssembly 的纯 JavaScript 解释器
- 完全隔离，无法访问 Node.js 原生 API
- 内置常用库 shim（crypto-js、lodash 等）
- 支持异步操作（async/await）
- 提供内置 `bru.sleep()` 方法

#### 2.2 Node VM 沙箱

**核心实现**：`runScriptInNodeVm()` (`packages/bruno-js/src/sandbox/node-vm/index.js:24`)

**特性**：
- 基于 Node.js 内置 `vm` 模块
- 支持 require() 加载本地模块（需配置允许路径）
- 可访问部分 Node.js API（白名单限制）
- 支持自定义 require 路径
- 提供更强的调试支持

### 三、上下文对象（原始）

沙箱执行环境中注入以下核心对象，供脚本访问和操作：

#### 3.1 Bru 对象（Bruno 核心 API）

**类定义**：`Bru` (`packages/bruno-js/src/bru.js:10`)

主要功能包括：变量管理（`getEnvVar`/`setEnvVar`/`getVar`/`setVar` 等）、流程控制（`runner.skipRequest`/`runner.stopExecution`/`runner.setNextRequest`）、工具方法（`sleep`/`sendRequest`）、Cookie 管理等。

#### 3.2 BrunoRequest 对象（请求对象）

**类定义**：`BrunoRequest` (`packages/bruno-js/src/bruno-request.js:3`)

提供 URL 操作、Header 操作、Body 操作、认证相关等 API。

#### 3.3 BrunoResponse 对象（响应对象）

**类定义**：`BrunoResponse`（类似 `BrunoRequest` 结构）

注意：可视化断言使用的不是此类，而是 `createResponseParser()` 返回的函数对象。

#### 3.4 测试与断言对象

包括 `test()` 函数包装器和完整的 chai 断言库支持。

### 四、断言判定链路（原始）

#### 4.1 可视化断言执行流程

核心方法：`AssertRuntime.runAssertions()`，使用 `createResponseParser()` 创建响应解析器。

#### 4.2 脚本内断言执行流程

核心方法：`TestRuntime.runTests()`，使用 `BrunoResponse` 类实例。

#### 4.3 自定义 Chai 断言扩展

包括 `json` 属性断言、`jsonSchema` 方法、`match` 方法、`jsonBody` 方法等。

---

## 关键调用链结论（原始）

### 请求前脚本执行链
```
ScriptRuntime.runRequestScript()
  ├─ new Bru() → 初始化变量上下文
  ├─ new BrunoRequest() → 包装请求对象
  ├─ createBruTestResultMethods() → 创建测试结果收集器
  ├─ 构建 context { bru, req, test, expect, assert, console, ... }
  └─ 选择沙箱执行
     ├─ QuickJS: executeQuickJsVmAsync()
     └─ NodeVM: runScriptInNodeVm()
```

### 请求后脚本执行链
```
ScriptRuntime.runResponseScript()
  ├─ new Bru() → 初始化变量上下文
  ├─ new BrunoRequest() → 包装请求对象
  ├─ new BrunoResponse() → 包装响应对象
  ├─ createBruTestResultMethods() → 创建测试结果收集器
  ├─ 构建 context { bru, req, res, test, expect, assert, console, ... }
  └─ 选择沙箱执行
     ├─ QuickJS: executeQuickJsVmAsync()
     └─ NodeVM: runScriptInNodeVm()
```

### 可视化断言执行链
```
AssertRuntime.runAssertions()
  ├─ new Bru() → 初始化变量上下文
  ├─ new BrunoRequest() → 包装请求对象
  ├─ createResponseParser() → 解析响应（函数式对象）
  ├─ 构建 context { bru, req, res, 所有层级变量 }
  └─ 遍历断言
     ├─ evaluateJsExpressionBasedOnRuntime() → LHS 求值
     ├─ parseAssertionOperator() → 操作符解析
     ├─ evaluateRhsOperand() → RHS 求值
     ├─ chai 断言执行
     └─ 结果收集
```

### 测试脚本执行链
```
TestRuntime.runTests()
  ├─ new Bru() → 初始化变量上下文
  ├─ new BrunoRequest() → 包装请求对象
  ├─ new BrunoResponse() → 包装响应对象
  ├─ createBruTestResultMethods() → 创建测试结果收集器
  ├─ 构建 context { test, bru, req, res, expect, assert, jwt, console, ... }
  └─ 选择沙箱执行
     ├─ QuickJS: executeQuickJsVmAsync()
     └─ NodeVM: runScriptInNodeVm()
```

### 核心设计原则
1. **双重沙箱保障**：QuickJS（安全优先）+ Node VM（灵活优先），用户可根据场景选择
2. **统一上下文注入**：无论哪种脚本类型，上下文对象 API 保持一致
3. **错误隔离**：脚本执行异常不会中断主流程，partialResults 确保已执行测试结果可追溯
4. **行号映射**：通过脚本包装偏移量计算，准确将沙箱错误映射到原始源文件行号
5. **渐进式断言**：可视化断言 + 脚本断言双路径，满足不同复杂度需求
