# Bruno CLI 测试运行器技术文档

## 概述

Bruno CLI 测试运行器是一个用于批量执行 API 集合测试的命令行工具，支持在 CI/CD 环境中自动化运行。本文档深入分析其核心架构、参数解析机制、断言执行流程、报告生成策略与退出码规范。

**核心入口**：`packages/bruno-cli/bin/bru.js` → `src/commands/run.js`

---

## 一、参数解析与运行计划

### 1.1 核心命令参数定义

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

### 1.2 环境变量加载优先级机制

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

### 1.3 运行计划构建

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

### 1.4 集合数据结构

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

## 二、断言执行机制

### 2.1 断言操作符完整列表

**源码位置**：`packages/bruno-js/src/runtime/assert-runtime.js:202-271`

断言操作符定义在 `value` 字段前缀，支持以下 30 种操作符：

| 分类 | 操作符 | 说明 | 示例写法 |
|-----|-------|------|---------|
| **等值比较** | `eq` | 等于 | `res.status eq 200` |
| | `neq` | 不等于 | `res.body.id neq 0` |
| **数值比较** | `gt` | 大于 | `res.body.total gt 100` |
| | `gte` | 大于等于 | `res.body.pages gte 5` |
| | `lt` | 小于 | `res.responseTime lt 1000` |
| | `lte` | 小于等于 | `res.responseTime lte 500` |
| | `between` | 区间内 | `res.status between 200, 299` |
| **包含匹配** | `in` | 在列表中 | `res.body.role in admin,moderator` |
| | `notIn` | 不在列表中 | `res.body.id notIn 0,-1` |
| | `contains` | 包含 | `res.body.name contains admin` |
| | `notContains` | 不包含 | `res.body.email notContains test` |
| **长度/类型** | `length` | 长度等于 | `res.body.items length 10` |
| | `isArray` | 是数组 | `res.body.items isArray` |
| | `isNumber` | 是数字 | `res.body.id isNumber` |
| | `isString` | 是字符串 | `res.body.name isString` |
| | `isBoolean` | 是布尔值 | `res.body.active isBoolean` |
| | `isJson` | 是 JSON 对象/数组 | `res.body isJson` |
| **正则/字符串** | `matches` | 匹配正则 | `res.body.uuid matches ^[a-f0-9]{32}$` |
| | `notMatches` | 不匹配正则 | `res.body.email notMatches ^test@` |
| | `startsWith` | 以...开头 | `res.body.name startsWith user_` |
| | `endsWith` | 以...结尾 | `res.body.email endsWith @example.com` |
| **空值判断** | `isEmpty` | 为空 | `res.body.error isEmpty` |
| | `isNotEmpty` | 不为空 | `res.body.data isNotEmpty` |
| **存在性判断** | `isNull` | 为 null | `res.body.deletedAt isNull` |
| | `isUndefined` | 为 undefined | `res.body.extra isUndefined` |
| | `isDefined` | 已定义 | `res.body.id isDefined` |
| **布尔判断** | `isTruthy` | 真值 | `res.body.success isTruthy` |
| | `isFalsy` | 假值 | `res.body.error isFalsy` |

### 2.2 真实执行路径

**运行时类**：`AssertRuntime`（packages/bruno-js/src/runtime/assert-runtime.js）

```javascript
// 断言执行 5 步法
runAssertions(assertions, request, response, envVariables, runtimeVariables, processEnvVars) {
  // 1. 构建执行上下文
  const bruContext = {
    bru: new Bru(...),      // Bruno API 对象
    req: new BrunoRequest(request),  // 请求对象
    res: createResponseParser(response) // 响应对象（支持 res.status/res.body/res.headers）
  };
  const context = {
    ...globalEnvironmentVariables,
    ...collectionVariables,
    ...envVariables,
    ...folderVariables,
    ...requestVariables,
    ...runtimeVariables,
    ...processEnvVars,
    ...bruContext
  };

  // 2. 解析断言操作符（从 value 字段前缀提取）
  const { operator, value: rhsOperand } = parseAssertionOperator(assertion.value);
  // 例如 value = "neq 200" → { operator: "neq", rhsOperand: "200" }

  // 3. LHS 求值：JS 表达式执行
  // lhsExpr = "res.body.items[0].id" → 在 context 中执行 JS 表达式
  const lhs = evaluateJsExpressionBasedOnRuntime(lhsExpr, context, this.runtime);

  // 4. RHS 求值：根据操作符类型特殊处理
  const rhs = evaluateRhsOperand(rhsOperand, operator, context, this.runtime);
  // - in/notIn: 拆分数组 → 每个元素作为模板字符串求值
  // - between: 拆分成 [min, max] → 分别求值
  // - matches/notMatches: 正则表达式字符串处理
  // - 一元操作符: 返回 undefined
  // - 其他: 作为 JS 模板字符串求值

  // 5. chai.js 断言执行
  switch (operator) {
    case 'eq': expect(lhs).to.equal(rhs); break;
    case 'neq': expect(lhs).to.not.equal(rhs); break;
    case 'gt': expect(lhs).to.be.greaterThan(rhs); break;
    case 'gte': expect(lhs).to.be.greaterThanOrEqual(rhs); break;
    case 'lt': expect(lhs).to.be.lessThan(rhs); break;
    case 'lte': expect(lhs).to.be.lessThanOrEqual(rhs); break;
    case 'in': expect(lhs).to.be.oneOf(rhs); break;
    case 'notIn': expect(lhs).to.not.be.oneOf(rhs); break;
    case 'contains': expect(lhs).to.include(rhs); break;
    case 'notContains': expect(lhs).to.not.include(rhs); break;
    case 'length': expect(lhs).to.have.lengthOf(rhs); break;
    case 'matches': expect(lhs).to.match(new RegExp(rhs)); break;
    case 'notMatches': expect(lhs).to.not.match(new RegExp(rhs)); break;
    case 'startsWith': expect(lhs).to.startWith(rhs); break;
    case 'endsWith': expect(lhs).to.endWith(rhs); break;
    case 'between': expect(lhs).to.be.within(rhs[0], rhs[1]); break;
    case 'isEmpty': expect(lhs).to.be.empty; break;
    case 'isNotEmpty': expect(lhs).to.not.be.empty; break;
    case 'isNull': expect(lhs).to.be.null; break;
    case 'isUndefined': expect(lhs).to.be.undefined; break;
    case 'isDefined': expect(lhs).to.not.be.undefined; break;
    case 'isTruthy': expect(lhs).to.be.true; break;
    case 'isFalsy': expect(lhs).to.be.false; break;
    case 'isJson': expect(lhs).to.be.json; break;
    case 'isNumber': expect(lhs).to.be.a('number'); break;
    case 'isString': expect(lhs).to.be.a('string'); break;
    case 'isBoolean': expect(lhs).to.be.a('boolean'); break;
    case 'isArray': expect(lhs).to.be.a('array'); break;
  }
}
```

### 2.3 单请求执行生命周期

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
│  - 基于 chai.js 的 30 种断言操作符                          │
│  - LHS/RHS 均为 JS 表达式求值                               │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                       阶段 7: 测试脚本                       │
│  TestRuntime.runTests()                                     │
│  - test() / expect() 链式调用                               │
│  - 异常: 添加 synthetic fail 结果                           │
└─────────────────────────────────────────────────────────────┘
```

### 2.4 脚本错误处理策略

| 阶段 | 错误处理行为 | 结果影响 |
|------|-------------|----------|
| **Pre-request** | 立即返回 `status: 'error'` | 请求不发送，后续流程终止 |
| **Post-response** | 记录错误 → 添加 synthetic fail → 继续 | 单请求标记为失败 |
| **Tests** | 记录错误 → 添加 synthetic fail → 继续 | 单请求标记为失败 |

### 2.5 执行结果数据结构

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
    {
      uid: "nanoid",
      lhsExpr: "res.status",   // 左操作数表达式
      rhsExpr: "eq 200",       // 原始右操作数字符串（含操作符）
      rhsOperand: "200",       // 提取操作符后的右操作数
      operator: "eq",          // 断言操作符
      status: "pass" | "fail",
      error: null | string     // chai 断言失败消息
    }
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

## 三、报告输出与退出码

### 3.1 报告生成流程

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

### 3.2 JUnit 报告格式

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

### 3.3 HTML 报告格式

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

### 3.4 汇总统计计算

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

### 3.5 敏感信息过滤

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

### 3.6 退出码规范

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

---

## 四、入口到退出码串联小结

### 4.1 完整执行路径概览

```
CLI 入口 (bin/bru.js)
    ↓
yargs 解析参数 (commands/run.js builder)
    ↓
handler() 主函数开始
    ├─ 验证集合根目录 → 非集合目录 → 退出码 4
    ├─ 加载环境变量 → 环境文件不存在 → 退出码 6
    ├─ 解析 --env-var 覆盖 → 格式错误 → 退出码 7/8
    ├─ 构建执行栈 getCallStack()
    └─ 循环执行每个请求:
         ├─ runSingleRequest()
         │   ├─ pre-request 脚本异常 → status: error
         │   ├─ 网络请求异常 → status: error
         │   ├─ post-response 脚本异常 → 添加 fail 结果
         │   ├─ Assertions 断言失败 → 添加 fail 结果
         │   └─ Tests 脚本异常 → 添加 fail 结果
         ├─ --bail 模式检测 → 任意失败立即终止循环
         └─ --delay 延迟控制
    ↓
getRunnerSummary(results) 统计汇总
    ↓
sanitizeResultsForReporter() 清理敏感数据
    ↓
生成报告 (JSON/JUnit/HTML)
    ├─ 输出目录不存在 → 退出码 2
    └─ 格式不支持 → 退出码 9
    ↓
判定最终退出码
    ├─ 任意失败/错误存在 → 退出码 1
    └─ 全部通过 → 退出码 0
    ↓
process.exit(exitCode)
```

### 4.2 关键设计决策

1. **CI友好的退出码策略**：所有失败（请求/断言/测试/脚本）统一返回码 1，便于 CI 流水线直接判定

2. **渐进式失败处理**：Pre-request 失败立即终止请求但不影响其他请求（除非 --bail）；Post-response/Tests 失败标记后继续执行，最大化测试覆盖率

3. **沙箱一致性**：断言 LHS/RHS 与测试脚本使用相同的 JS 沙箱（quickjs/nodevm），保证求值逻辑一致

4. **报告可组合性**：支持多格式报告并行输出，满足不同 CI 系统集成需求（JUnit 用于 CI 平台，HTML 用于人工查看，JSON 用于二次处理）

5. **无限循环保护**：`bru.nextRequest()` 跳转计数超过 10000 次触发退出码 3，防止测试脚本死循环导致 CI 挂起
