# Bruno CLI 测试运行器技术文档

## 概述

Bruno CLI 测试运行器是一个用于批量执行 API 集合测试的命令行工具，支持在 CI/CD 环境中自动化运行。本文档深入分析其核心架构、参数解析机制、断言执行流程、报告生成策略与退出码规范。

**核心入口**：`packages/bruno-cli/bin/bru.js` → `src/commands/run.js`

---

## 一、整体架构

### 1.1 核心模块分层

```
┌─────────────────────────────────────────────────────────────┐
│                     CLI 参数解析层                           │
│  packages/bruno-cli/src/commands/run.js                     │
│  - yargs 参数定义与示例                                     │
│  - 环境变量/输出格式配置                                    │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                     集合加载与运行计划层                    │
│  packages/bruno-cli/src/utils/collection.js                 │
│  - createCollectionJsonFromPathname()                       │
│  - getCallStack() - 构建请求执行栈                          │
│  - mergeHeaders/mergeVars/mergeScripts/mergeAuth            │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                     单请求运行器层                           │
│  packages/bruno-cli/src/runner/run-single-request.js        │
│  - 预处理脚本执行 → 发送请求 → 后置脚本 → 断言 → 测试      │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                     脚本/断言运行时层                        │
│  packages/bruno-js/src/runtime/                             │
│  - ScriptRuntime (pre-request / post-response)              │
│  - TestRuntime / AssertRuntime / VarsRuntime                │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                     报告输出层                               │
│  packages/bruno-cli/src/reporters/                          │
│  - html.js  HTML 报告生成                                   │
│  - junit.js JUnit XML 报告                                  │
│  - JSON 原生格式                                            │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 执行流程总览

```
1. 参数解析 (yargs)
   ↓
2. 集合加载 (createCollectionJsonFromPathname)
   ↓
3. 环境变量处理 (--env / --env-file / --env-var)
   ↓
4. 构建执行栈 (getCallStack)
   ├─ 按路径/文件夹筛选请求
   ├─ 递归遍历子目录
   ├─ 标签过滤 (--tags / --exclude-tags)
   └─ 测试-only 过滤 (--tests-only)
   ↓
5. 循环执行每个请求 (runSingleRequest)
   ├─ Pre-request 脚本
   ├─ 变量插值
   ├─ 发送 HTTP 请求
   ├─ Post-response 脚本
   ├─ 断言验证
   └─ 测试脚本
   ↓
6. 汇总执行结果 (getRunnerSummary)
   ↓
7. 生成报告 (多格式并行)
   ↓
8. 退出码返回
```

---

## 二、参数解析与运行计划

### 2.1 核心命令参数定义

**文件**：`packages/bruno-cli/src/commands/run.js:112-306`

| 参数分类 | 关键参数 | 类型 | 说明 |
|---------|---------|------|------|
| **执行范围** | `paths` | array | 请求文件/文件夹路径，默认当前目录 |
| | `-r` | boolean | 递归运行子文件夹 |
| | `--tags` | string | 按包含标签过滤，逗号分隔 |
| | `--exclude-tags` | string | 按排除标签过滤 |
| | `--tests-only` | boolean | 仅运行含测试/断言的请求 |
| **环境配置** | `--env` | string | 指定环境名称 |
| | `--env-file` | string | 环境文件路径 (.bru/.json/.yml) |
| | `--global-env` | string | 全局环境名称 (需 workspace) |
| | `--env-var` | string | 覆盖单个变量，可多次使用 |
| **执行控制** | `--delay` | number | 请求间延迟 (ms) |
| | `--bail` | boolean | 失败即终止 |
| | `--sandbox` | string | JS 沙箱模式: safe(默认)/developer |
| **安全配置** | `--insecure` | boolean | 允许不安全 TLS 连接 |
| | `--cacert` | string | 自定义 CA 证书 |
| | `--ignore-truststore` | boolean | 忽略默认信任存储 |
| | `--client-cert-config` | string | 客户端证书配置 |
| **代理配置** | `--noproxy` | boolean | 禁用所有代理 |
| | `--cache-ssl-session` | boolean | 启用 SSL 会话缓存 |
| **输出配置** | `-o/--output` | string | 输出文件路径 |
| | `--format` | string | 输出格式: json/junit/html |
| | `--reporter-json` | string | JSON 报告路径 |
| | `--reporter-junit` | string | JUnit 报告路径 |
| | `--reporter-html` | string | HTML 报告路径 |
| **敏感信息** | `--reporter-skip-all-headers` | boolean | 报告中跳过所有头 |
| | `--reporter-skip-headers` | array | 跳过指定头 |
| | `--reporter-skip-body` | boolean | 跳过请求/响应体 |

### 2.2 环境变量加载优先级机制

**代码位置**：`packages/bruno-cli/src/commands/run.js:383-525`

```
优先级由高到低：

1. --env-var 命令行覆盖
   格式: name=value，可多次指定
   代码: envVar → split → envVars[match[1]] = match[2]

2. --env-file 指定的环境文件
   支持格式: .bru / .json / .yml
   解析: loadEnvFromFile() → parseEnvironment()

3. --env 集合内命名环境
   路径: environments/{name}.{ext}
   会与 --env-file 合并（后加载的覆盖先加载）

4. --global-env 全局环境
   需 workspace.yml 定位工作空间
   路径: {workspace}/environments/{globalEnv}.yml

5. 集合根目录 .env 文件
   自动加载，注入 processEnvVars

6. 系统环境变量 process.env
   可通过 {{process.env.VAR_NAME}} 访问
```

### 2.3 运行计划构建

**核心函数**：`getCallStack(resolvedPaths, collection, { recursive })`

**文件**：`packages/bruno-cli/src/utils/collection.js:490-520`

```javascript
// 执行栈构建逻辑
function getCallStack(resolvedPaths, collection, { recursive }) {
  let requestItems = [];

  for (const resolvedPath of resolvedPaths) {
    // 1. 集合根目录：获取所有请求
    if (resolvedPath === collection.pathname) {
      requestItems = requestItems.concat(
        getAllRequestsInFolder(collection.items, recursive)
      );
      continue;
    }

    // 2. 查找路径对应项
    const item = findItemInCollection(collection, resolvedPath);

    // 3. 文件夹：获取文件夹内所有请求
    if (item.type === 'folder') {
      requestItems = requestItems.concat(
        getAllRequestsInFolder(item.items, recursive)
      );
    }
    // 4. 单个请求：直接加入
    else {
      requestItems.push(item);
    }
  }

  return requestItems;
}
```

**请求过滤管道**：

```javascript
// packages/bruno-cli/src/commands/run.js:622-639

// 1. --tests-only 过滤
if (testsOnly) {
  requestItems = requestItems.filter((item) => {
    const hasTests = hasExecutableTestInScript(item.request?.tests);
    const hasAssertions = item.request?.assertions.some(x => x.enabled);
    const hasPreRequestTests = hasExecutableTestInScript(item.request?.script?.req);
    const hasPostResponseTests = hasExecutableTestInScript(item.request?.script?.res);
    return hasTests || hasAssertions || hasPreRequestTests || hasPostResponseTests;
  });
}

// 2. 标签过滤
requestItems = requestItems.filter((item) => {
  return isRequestTagsIncluded(item.tags, includeTags, excludeTags);
});
```

### 2.4 集合数据结构

**集合加载函数**：`createCollectionJsonFromPathname(collectionPath)`

```javascript
// 输出结构
{
  brunoConfig: {           // bruno.json 配置
    version: "1",
    name: "collection-name",
    type: "collection"
  },
  format: "bru" | "yml",  // 集合格式
  root: {},               // 集合级脚本/头/变量
  pathname: "/path/to/collection",
  items: [                // 扁平化后的请求/文件夹树
    {
      type: "folder" | "http-request" | "graphql-request",
      name: "item-name",
      pathname: "/full/path",
      seq: 1,             // 排序序号
      // folder 特有:
      items: [...],       // 子项
      root: {...},        // 文件夹级配置
      // request 特有:
      request: {
        method: "GET",
        url: "...",
        headers: [...],
        script: { req: "...", res: "..." },
        tests: "...",
        assertions: [...],
        vars: { req: [...], res: [...] }
      }
    }
  ]
}
```

---

## 三、断言执行与测试运行

### 3.1 单请求执行生命周期

**核心函数**：`runSingleRequest()`

**文件**：`packages/bruno-cli/src/runner/run-single-request.js:86-958`

```
┌─────────────────────────────────────────────────────────────┐
│                        阶段 1: 预处理                        │
│  1. prepareRequest() 合并继承头/变量/脚本/认证              │
│  2. extractPromptVariablesForRequest() 检测提示变量         │
│     → 含 {{prompt:...}} 则标记 skipped 并返回                │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   阶段 2: Pre-request 脚本                   │
│  ScriptRuntime.runRequestScript()                           │
│  - 运行 bru.setVar() / bru.getVar()                         │
│  - 支持 bru.skipRequest() / bru.nextRequest()               │
│  - 支持 bru.stopExecution()                                 │
│  - 异常: 返回 status: 'error' 并终止后续请求流程             │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                      阶段 3: 变量插值                        │
│  interpolateVars() 替换 {{variable}} 语法                   │
│  - 优先级: runtime > request > folder > collection > env    │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                      阶段 4: 发送请求                        │
│  1. TLS/代理/证书配置                                       │
│  2. OAuth1/OAuth2/Digest/AWSv4/NTLM 认证                    │
│  3. Cookie 自动管理                                         │
│  4. axiosInstance(request)                                  │
│  - 网络异常: 返回 status: 'error'                           │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                  阶段 5: Post-response 脚本                  │
│  ScriptRuntime.runResponseScript()                          │
│  - 可访问 res.status / res.body / res.headers               │
│  - 异常: 添加 synthetic fail 结果，继续执行                  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                       阶段 6: 断言执行                       │
│  AssertRuntime.runAssertions()                              │
│  - 纯数据驱动，无需脚本                                     │
│  - 支持: status code / header / body / time 等断言          │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                       阶段 7: 测试脚本                       │
│  TestRuntime.runTests()                                     │
│  - test() / expect() 链式调用                               │
│  - 异常: 添加 synthetic fail 结果                           │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 断言执行机制

**断言结构**（`item.request.assertions`）：

```javascript
[
  {
    "enabled": true,
    "name": "res.status",
    "value": "200",
    "assertion": "eq"
  },
  {
    "enabled": true,
    "name": "res.body.id",
    "value": "123",
    "assertion": "ne"
  }
]
```

**断言执行器**：`packages/bruno-js/src/runtime/assert-runtime.js`

```javascript
class AssertRuntime {
  runAssertions(
    assertions,
    request,
    response,
    envVariables,
    runtimeVariables,
    processEnvVars
  ) {
    return assertions.map((assertion) => {
      if (!assertion.enabled) return null;

      // 1. 插值左右表达式
      const lhs = interpolateString(assertion.name, context);
      const rhs = interpolateString(assertion.value, context);

      // 2. 计算左值（支持 JSONPath）
      const lhsValue = evaluate(lhs, { req: request, res: response });

      // 3. 根据断言类型执行比较
      const result = executeAssertion(assertion.assertion, lhsValue, rhs);

      return {
        status: result ? "pass" : "fail",
        lhsExpr: assertion.name,
        lhsValue: lhsValue,
        rhsExpr: assertion.value,
        rhsValue: rhs,
        error: result ? null : `Expected ${rhs}, got ${lhsValue}`
      };
    }).filter(Boolean);
  }
}
```

### 3.3 测试脚本执行机制

**测试运行时**：`packages/bruno-js/src/runtime/test-runtime.js`

```javascript
class TestRuntime {
  async runTests(
    testScript,
    request,
    response,
    envVariables,
    runtimeVariables,
    collectionPath,
    onConsoleLog,
    processEnvVars,
    scriptingConfig,
    runRequestCallback,
    collectionName
  ) {
    const results = [];

    // 注入测试 API
    const test = (description, callback) => {
      try {
        callback();
        results.push({ status: "pass", description });
      } catch (error) {
        results.push({
          status: "fail",
          description,
          error: error.message,
          stack: error.stack
        });
      }
    };

    // 注入 expect API
    const expect = (actual) => createExpectChain(actual, results);

    // 在沙箱中执行脚本
    const sandbox = createSandbox({
      test,
      expect,
      req: request,
      res: response,
      bru: createBruApi(...),
      console: createConsoleProxy(onConsoleLog)
    });

    await sandbox.run(testScript);

    return { results };
  }
}
```

**脚本错误处理策略**：

| 阶段 | 错误处理行为 | 结果影响 |
|------|-------------|----------|
| **Pre-request** | 立即返回 `status: 'error'` | 请求不发送，后续流程终止 |
| **Post-response** | 记录错误 → 添加 synthetic fail → 继续 | 单请求标记为失败 |
| **Tests** | 记录错误 → 添加 synthetic fail → 继续 | 单请求标记为失败 |

### 3.4 执行结果数据结构

```javascript
// 单个请求执行结果
{
  test: { filename: "relative/path.bru" },
  request: { method, url, headers, data },
  response: {
    status: 200,
    statusText: "OK",
    headers: {},
    data: {},
    responseTime: 123, // ms
    duration: 123,
    size: 456
  },
  error: null | string,
  status: "pass" | "fail" | "error" | "skipped",
  skipped: boolean,

  // 四类测试结果
  assertionResults: [          // 数据驱动断言
    { status: "pass" | "fail", lhsExpr, rhsExpr, error }
  ],
  preRequestTestResults: [     // pre-request 脚本中的测试
    { status, description, error, stack }
  ],
  postResponseTestResults: [   // post-response 脚本中的测试
    { status, description, error, stack }
  ],
  testResults: [               // tests 脚本中的测试
    { status, description, error, stack }
  ],

  // 流控制
  nextRequestName: string | null,  // bru.nextRequest()
  shouldStopRunnerExecution: boolean,  // bru.stopExecution()

  // 元数据
  runDuration: 0.456,          // 秒
  suitename: "folder/request",
  name: "request-name",
  path: "relative/path"
}
```

---

## 四、报告输出与退出码

### 4.1 报告生成流程

**代码位置**：`packages/bruno-cli/src/commands/run.js:778-816`

```javascript
// 多报告格式支持
const formats = {};

// 兼容模式: --output + --format
if (outputPath && outputPath.length) {
  formats[format] = outputPath;
}

// 多报告并行模式: 可同时生成多种格式
if (reporterHtml) formats['html'] = reporterHtml;
if (reporterJson) formats['json'] = reporterJson;
if (reporterJunit) formats['junit'] = reporterJunit;

// 报告生成器映射
const reporters = {
  json: (path) => fs.writeFileSync(path, JSON.stringify({ summary, results }, null, 2)),
  junit: (path) => makeJUnitOutput(results, path),
  html: (path) => makeHtmlOutput({ summary, results }, path, timestamp, envName)
};

// 执行报告生成
for (const formatter of Object.keys(formats)) {
  const reportPath = formats[formatter];
  const reporter = reporters[formatter];

  // 验证输出目录存在
  const outputDir = path.dirname(reportPath);
  if (!await exists(outputDir)) {
    process.exit(EXIT_STATUS.ERROR_MISSING_OUTPUT_DIR);
  }

  reporter(reportPath);
  console.log(`Wrote ${formatter} results to ${reportPath}`);
}
```

### 4.2 JUnit 报告格式

**文件**：`packages/bruno-cli/src/reporters/junit.js`

```xml
<!-- 输出结构 -->
<testsuites>
  <testsuite
    name="请求名称"
    file="文件路径"
    errors="0"
    failures="失败数"
    skipped="0"
    tests="总测试数"
    timestamp="ISO时间"
    hostname="主机名"
    time="执行时间(秒)">

    <!-- 每个断言/测试对应一个 testcase -->
    <testcase
      name="断言表达式 / 测试描述"
      status="pass|fail"
      classname="请求URL"
      time="分摊时间">
      <failure type="failure" message="错误信息" />
    </testcase>

  </testsuite>
</testsuites>
```

**测试用例计数规则**：

```
总测试数 = assertionResults.length （断言）
        + preRequestTestResults.length （前置测试）
        + testResults.length （测试脚本）
        + postResponseTestResults.length （后置测试）
```

### 4.3 HTML 报告格式

**文件**：`packages/bruno-cli/src/reporters/html.js` → 调用 `@usebruno/common/runner`

```javascript
// HTML 报告输入结构
{
  runnerResults: [{
    iterationIndex: 0,      // 迭代序号（集合运行器支持多迭代）
    results: [...],         // 请求结果数组
    summary: {...}          // 该迭代汇总
  }],
  version: "usebruno v1.x",
  environment: "环境名称",
  runCompletionTime: "ISO 时间戳"
}
```

### 4.4 汇总统计计算

**函数**：`getRunnerSummary(results)`

**文件**：`packages/bruno-common/src/runner/runner-summary.ts`

```javascript
// 汇总统计维度
{
  // 请求级统计
  totalRequests: number,     // 总请求数
  passedRequests: number,    // 完全通过（所有测试/断言通过且无错误）
  failedRequests: number,    // 有测试/断言失败
  errorRequests: number,     // 请求错误（网络异常/脚本异常等）
  skippedRequests: number,   // 跳过请求

  // 断言级统计
  totalAssertions: number,   // 总断言数
  passedAssertions: number,  // 通过断言数
  failedAssertions: number,  // 失败断言数

  // 测试脚本级统计
  totalTests: number,        // 总测试数（tests 脚本）
  passedTests: number,
  failedTests: number,

  // Pre-request 测试统计
  totalPreRequestTests: number,
  passedPreRequestTests: number,
  failedPreRequestTests: number,

  // Post-response 测试统计
  totalPostResponseTests: number,
  passedPostResponseTests: number,
  failedPostResponseTests: number
}
```

**请求状态判定逻辑**：

```javascript
// 只要有任何一项失败，该请求即标记为失败
let anyFailed =
  testResults.some(r => r.status === 'fail') ||
  assertionResults.some(r => r.status === 'fail') ||
  preRequestTestResults.some(r => r.status === 'fail') ||
  postResponseTestResults.some(r => r.status === 'fail');

if (!anyFailed && status !== 'error') {
  passedRequests++;
} else if (anyFailed) {
  failedRequests++;
} else {
  errorRequests++;
}
```

### 4.5 退出码规范

**文件**：`packages/bruno-cli/src/constants.js:6-35`

| 退出码 | 常量名 | 触发条件 |
|-------|--------|---------|
| **0** | - | 所有请求成功，无失败/错误 |
| **1** | `ERROR_FAILED_COLLECTION` | 存在失败的请求/测试/断言 <br> `failed* + errorRequests > 0` |
| **2** | `ERROR_MISSING_OUTPUT_DIR` | 报告输出目录不存在 |
| **3** | `ERROR_INFINITE_LOOP` | 检测到无限跳转 <br> `bru.nextRequest()` 超过 10000 次 |
| **4** | `ERROR_NOT_IN_COLLECTION` | 当前目录不是 Bruno 集合 <br> （无 bruno.json / opencollection.yml） |
| **5** | `ERROR_FILE_NOT_FOUND` | 指定的请求/文件夹路径不存在 |
| **6** | `ERROR_ENV_NOT_FOUND` | 指定的环境文件不存在 |
| **7** | `ERROR_MALFORMED_ENV_OVERRIDE` | `--env-var` 格式错误 |
| **8** | `ERROR_INCORRECT_ENV_OVERRIDE` | `--env-var` 无法解析 |
| **9** | `ERROR_INCORRECT_OUTPUT_FORMAT` | 不支持的输出格式 |
| **10** | `ERROR_INVALID_FILE` | 文件解析失败（JSON/YAML/Bru 语法错误） |
| **11** | `ERROR_WORKSPACE_NOT_FOUND` | Workspace 目录不存在 |
| **12** | `ERROR_GLOBAL_ENV_REQUIRES_WORKSPACE` | 全局环境需要 workspace |
| **13** | `ERROR_GLOBAL_ENV_NOT_FOUND` | 指定的全局环境不存在 |
| **255** | `ERROR_GENERIC` | 其他未分类异常 |

**退出码计算逻辑**：

```javascript
// packages/bruno-cli/src/commands/run.js:818-820

// 有任何失败或错误 → 退出码 1
if (
  summary.failedAssertions +
  summary.failedTests +
  summary.failedPreRequestTests +
  summary.failedPostResponseTests +
  summary.failedRequests > 0 ||
  summary.errorRequests > 0
) {
  process.exit(EXIT_STATUS.ERROR_FAILED_COLLECTION);
}

// 否则正常退出（退出码 0）
```

### 4.6 敏感信息过滤

**代码位置**：`packages/bruno-cli/src/commands/run.js:720-725`

```javascript
// 报告生成前清理敏感信息
sanitizeResultsForReporter(results, {
  skipAllHeaders: reporterSkipAllHeaders,       // 删除所有头
  skipHeaders: reporterSkipHeaders,             // 删除指定头
  skipRequestBody: reporterSkipRequestBody || reporterSkipBody,
  skipResponseBody: reporterSkipResponseBody || reporterSkipBody
});
```

---

## 附录：关键调用链

### 命令入口

```
bru run collection
  → handler() (run.js)
    → createCollectionJsonFromPathname()
    → 环境变量加载
    → getCallStack() 构建执行计划
    → forEach requestItem:
        → runSingleRequest()
          → prepareRequest()
          → ScriptRuntime.runRequestScript()
          → interpolateVars()
          → axiosInstance(request)
          → ScriptRuntime.runResponseScript()
          → AssertRuntime.runAssertions()
          → TestRuntime.runTests()
    → getRunnerSummary(results)
    → 生成报告
    → process.exit(exitCode)
```

### CI/CD 集成示例

```bash
# 基本用法：运行集合并生成 JUnit 报告
bru run --env staging --reporter-junit report.xml

# 多报告输出：同时生成 JSON、JUnit、HTML
bru run --reporter-json result.json \
        --reporter-junit junit.xml \
        --reporter-html report.html

# 安全模式：隐藏敏感信息
bru run --reporter-skip-all-headers \
        --reporter-skip-body

# 失败即终止：适合快速反馈流水线
bru run --bail --env production

# GitLab CI 配置示例: .gitlab-ci.yml
# api_test:
#   script:
#     - npm install -g @usebruno/cli
#     - bru run --env ci --reporter-junit junit-report.xml
#   artifacts:
#     reports:
#       junit: junit-report.xml
#     when: always
```
