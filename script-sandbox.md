# Bruno 脚本沙箱机制分析报告

## 一、脚本注入时机

Bruno 支持三种脚本执行时机，分别在请求生命周期的不同阶段注入执行：

### 1.1 请求前脚本（Pre-request Script）

**注入时机**：请求发送前执行，用于修改请求参数、设置变量等。

**调用入口**：`ScriptRuntime.runRequestScript()` (`packages/bruno-js/src/runtime/script-runtime.js:18`)

**执行流程**：
1. 构建 Bru 上下文对象，包含所有层级的变量
2. 构建 BrunoRequest 对象，暴露请求属性和方法
3. 注入 chai 断言库和 test 结果收集方法
4. 根据 runtime 配置选择沙箱执行
5. 执行完成后返回更新后的变量和测试结果

### 1.2 请求后脚本（Post-response Script）

**注入时机**：收到响应后执行，用于处理响应数据、断言验证等。

**调用入口**：`ScriptRuntime.runResponseScript()` (`packages/bruno-js/src/runtime/script-runtime.js:151`)

**执行流程**：
1. 构建 Bru 上下文对象（同请求前）
2. 构建 BrunoRequest 和 BrunoResponse 对象
3. 注入 chai 断言库和 test 结果收集方法
4. 根据 runtime 配置选择沙箱执行
5. 执行完成后返回更新后的变量和测试结果

### 1.3 自定义断言（Custom Assertions）

**注入时机**：响应返回后，与请求后脚本独立执行，用于可视化断言配置。

**调用入口**：`AssertRuntime.runAssertions()` (`packages/bruno-js/src/runtime/assert-runtime.js:408`)

**执行流程**：
1. 构建 Bru 上下文对象
2. 构建 BrunoRequest 和 BrunoResponse 对象
3. 遍历所有启用的断言
4. 解析断言操作符（eq、neq、gt、gte 等 20+ 种）
5. 在沙箱中执行表达式求值
6. 使用 chai 断言库进行验证
7. 收集断言结果（pass/fail）

---

## 二、运行沙箱

Bruno 提供两种沙箱运行模式，可在安全与灵活性之间权衡。

### 2.1 QuickJS 沙箱（默认）

**核心实现**：`executeQuickJsVmAsync()` (`packages/bruno-js/src/sandbox/quickjs/index.js:92`)

**特性**：
- 基于 WebAssembly 的纯 JavaScript 解释器
- 完全隔离，无法访问 Node.js 原生 API
- 内置常用库 shim（crypto-js、lodash 等）
- 支持异步操作（async/await）
- 提供内置 `bru.sleep()` 方法

**脚本包装**：
```javascript
// QUICKJS_SCRIPT_PREFIX + 用户脚本 + QUICKJS_SCRIPT_SUFFIX
(async () => {
  const setTimeout = async(fn, timer) => {
    v = await bru.sleep(timer);
    fn.apply();
  }
  await bru.sleep(0);
  try {
    // 用户脚本在此执行
  }
  catch(error) {
    throw error;
  }
  return 'done';
})()
```

**关键调用链**：
```
executeQuickJsVmAsync()
  ├─ loader() → 加载 QuickJS WASM 模块
  ├─ addCryptoUtilsShimToContext() → 注入 crypto 工具
  ├─ vm.evalCode(bundledCode) → 执行预打包库
  ├─ addBruShimToContext() → 注入 bru 对象
  ├─ addBrunoRequestShimToContext() → 注入 req 对象
  ├─ addBrunoResponseShimToContext() → 注入 res 对象
  ├─ addConsoleShimToContext() → 注入 console
  ├─ addLibraryShimsToContext() → 注入常用库
  ├─ addTestShimToContext() → 注入 test 函数
  ├─ wrapScriptInClosure() → 包装用户脚本
  └─ vm.evalCode(script) → 在沙箱中执行
```

### 2.2 Node VM 沙箱

**核心实现**：`runScriptInNodeVm()` (`packages/bruno-js/src/sandbox/node-vm/index.js:24`)

**特性**：
- 基于 Node.js 内置 `vm` 模块
- 支持 require() 加载本地模块（需配置允许路径）
- 可访问部分 Node.js API（白名单限制）
- 支持自定义 require 路径
- 提供更强的调试支持

**安全机制**：
- `vm.createContext()` 创建真正隔离的上下文
- `safeGlobals` 白名单控制可访问的全局对象
- `additionalContextRoots` 控制模块加载路径
- TypedArray 注入确保 API 兼容性

**脚本包装**：
```javascript
// NODEVM_SCRIPT_PREFIX + 用户脚本 + NODEVM_SCRIPT_SUFFIX
(async function(){
  // 用户脚本在此执行
})();
```

**关键调用链**：
```
runScriptInNodeVm()
  ├─ buildScriptContext() → 构建执行上下文
  │   ├─ 注入 bru、req、res、console 等
  │   ├─ 注入 safeGlobals 白名单中的 API
  │   └─ mixinTypedArrays() → 注入 TypedArray 构造函数
  ├─ vm.createContext() → 创建隔离上下文
  ├─ createCustomRequire() → 创建自定义 require 函数
  ├─ wrapScriptInClosure() → 包装用户脚本
  ├─ new vm.Script() → 编译脚本
  ├─ Error.prepareStackTrace → 捕获调用栈信息
  └─ compiledScript.runInContext() → 执行脚本
```

---

## 三、上下文对象

沙箱执行环境中注入以下核心对象，供脚本访问和操作。

### 3.1 bru 对象（Bruno 核心 API）

**类定义**：`Bru` (`packages/bruno-js/src/bru.js:10`)

**主要功能**：
- **变量管理**：
  - `getEnvVar(key)` / `setEnvVar(key, value, options)` - 环境变量
  - `getVar(key)` / `setVar(key, value)` - 运行时变量
  - `getGlobalEnvVar(key)` / `setGlobalEnvVar(key, value)` - 全局变量
  - `getCollectionVar(key)` / `getFolderVar(key)` / `getRequestVar(key)` - 各级变量
  - `interpolate(strOrObj)` - 变量插值

- **流程控制**：
  - `runner.skipRequest()` - 跳过当前请求
  - `runner.stopExecution()` - 停止集合执行
  - `runner.setNextRequest(name)` - 设置下一个执行请求
  - `bru.runRequest(pathname)` - 执行其他请求（需注入）

- **工具方法**：
  - `sleep(ms)` - 休眠
  - `sendRequest(options)` - 发送 HTTP 请求
  - `cwd()` - 获取集合路径
  - `getEnvName()` - 获取当前环境名
  - `resetOauth2Credential(credentialId)` - 重置 OAuth2 凭证
  - `utils.minifyJson()` / `utils.minifyXml()` - 格式处理

- **Cookie 管理**：
  - `cookies.get(name)` - 获取 Cookie
  - `cookies.set(name, value, options)` - 设置 Cookie
  - `cookies.list` - 列出所有 Cookie

### 3.2 req 对象（请求对象）

**类定义**：`BrunoRequest` (`packages/bruno-js/src/bruno-request.js:3`)

**主要 API**：
- **属性访问**：`req.url`、`req.method`、`req.headers`、`req.body`、`req.timeout`、`req.name`
- **URL 操作**：`getUrl()`、`setUrl()`、`getHost()`、`getPath()`、`getQueryString()`
- **方法操作**：`getMethod()`、`setMethod()`
- **Header 操作**：`getHeaders()`、`setHeaders()`、`getHeader(name)`、`setHeader(name, value)`、`deleteHeader(name)`、`headerList`
- **Body 操作**：`getBody(options)`、`setBody(data, options)`
- **认证相关**：`getAuthMode()`
- **其他**：`setTimeout()`、`setMaxRedirects()`、`onFail(callback)`、`getPathParams()`、`getTags()`

### 3.3 res 对象（响应对象）

**类定义**：`BrunoResponse`（类似 `BrunoRequest` 结构）

**主要 API**：
- `res.status` - 响应状态码
- `res.statusText` - 响应状态文本
- `res.headers` - 响应头
- `res.body` - 响应体（JSON 自动解析）
- `res.responseTime` - 响应时间
- `res.getHeader(name)` - 获取指定响应头
- `res.getBody(options)` - 获取响应体（支持 raw 模式）

### 3.4 测试与断言对象

**test 函数**（`packages/bruno-js/src/utils/results.js` 中创建）：
```javascript
test("测试名称", () => {
  // 断言逻辑
  expect(res.status).to.equal(200);
});
```

**chai 断言库**：
- `expect` - BDD 风格断言
- `assert` - TDD 风格断言
- 内置扩展：`json` 属性断言、`jsonSchema` 方法、`match` 正则匹配、`jsonBody` 路径断言

### 3.5 其他上下文

- `console` - 自定义日志输出（log、info、warn、debug、error）
- `jwt` - jsonwebtoken 库（仅在 TestRuntime 中注入）
- 各层级变量直接注入上下文（可直接访问变量名）

---

## 四、断言判定链路

### 4.1 可视化断言执行流程

**核心方法**：`AssertRuntime.runAssertions()` (`packages/bruno-js/src/runtime/assert-runtime.js:408`)

**完整链路**：
```
1. 过滤启用的断言
   ↓
2. 构建上下文（bru、req、res + 所有层级变量）
   ↓
3. 遍历每个断言项
   ├─ 解析左侧表达式（LHS）
   │   └─ evaluateJsExpressionBasedOnRuntime() → 在沙箱中求值
   ├─ 解析操作符和右侧操作数（RHS）
   │   └─ parseAssertionOperator() → 识别 20+ 种操作符
   ├─ 求值右侧操作数
   │   ├─ 普通值：直接求值
   │   ├─ between：解析为 [min, max]
   │   ├─ in/notIn：解析为数组
   │   └─ matches/notMatches：正则表达式处理
   ├─ 执行 chai 断言
   │   ├─ eq → expect(lhs).to.equal(rhs)
   │   ├─ neq → expect(lhs).to.not.equal(rhs)
   │   ├─ gt/gte → greaterThan/greaterThanOrEqual
   │   ├─ lt/lte → lessThan/lessThanOrEqual
   │   ├─ contains/notContains → include/notInclude
   │   ├─ length → have.lengthOf
   │   ├─ startsWith/endsWith → 字符串前缀/后缀
   │   ├─ between → within(min, max)
   │   ├─ isEmpty/isNotEmpty → empty/not.empty
   │   ├─ isNull/isUndefined/isDefined → null/undefined
   │   ├─ isTruthy/isFalsy → true/false
   │   ├─ isJson → 自定义 JSON 类型断言
   │   ├─ isNumber/isString/isBoolean/isArray → 类型断言
   │   └─ 其他操作符映射...
   └─ 收集结果
       ├─ 成功：{ status: 'pass', lhsExpr, rhsExpr, operator }
       └─ 失败：{ status: 'fail', error: err.message, ... }
   ↓
4. 将结果附加到 request.assertionResults
   ↓
5. 返回断言结果数组
```

### 4.2 脚本内断言执行流程

**核心方法**：`TestRuntime.runTests()` (`packages/bruno-js/src/runtime/test-runtime.js:17`)

**完整链路**：
```
1. 构建 Bru、BrunoRequest、BrunoResponse 对象
   ↓
2. 创建测试结果收集器 __brunoTestResults
   ↓
3. 创建 test() 函数包装器
   ├─ 接收测试名称和回调函数
   ├─ 执行回调中的断言
   ├─ 捕获异常（断言失败）
   └─ 记录测试结果（pass/fail）
   ↓
4. 注入 chai.expect、chai.assert、jwt 等
   ↓
5. 根据 runtime 选择沙箱执行测试脚本
   ↓
6. 收集所有测试结果
   ↓
7. 返回包含变量更新和测试结果的对象
```

### 4.3 自定义 Chai 断言扩展

**内置扩展**（`packages/bruno-js/src/runtime/assert-runtime.js:14-69`）：

1. **`json` 属性断言**：
   ```javascript
   expect(obj).to.be.json;  // 验证是 JSON 对象或数组
   ```

2. **`jsonSchema` 方法**：
   ```javascript
   expect(data).to.jsonSchema(schema);  // 验证 JSON Schema 合规性
   ```
   - 支持 Draft-07 版本
   - 支持自定义 Ajv 配置选项

3. **`match` 方法**：
   ```javascript
   expect(str).to.match(/regex/);  // 正则匹配
   ```

4. **`jsonBody` 方法**：
   ```javascript
   expect(body).to.jsonBody();  // 验证是 JSON body
   expect(body).to.jsonBody({ key: 'value' });  // 深度相等
   expect(body).to.jsonBody('data.items[0].name');  // 路径存在
   expect(body).to.jsonBody('data.items[0].name', 'expected');  // 路径值相等
   ```
   - 支持点号路径：`a.b.c`
   - 支持数组索引：`items[0]`
   - 支持引号包裹的键名：`data["a.b"]`
   - 支持嵌套组合

---

## 关键调用链结论

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
  ├─ createResponseParser() → 解析响应
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
