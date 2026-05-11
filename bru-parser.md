# Bru 文件解析机制详解

## 1. 概述

Bru 是 Bruno API 客户端中用于存储 HTTP 请求定义的纯文本文件格式。解析器负责将 Bru 纯文本文件转换为可执行的 JSON 结构，支持发送 HTTP 请求、认证、脚本执行等操作。

## 2. Bru 文件语法

Bru 文件经历了两个主要版本：V1 和 V2。

### 2.1 V1 格式（旧版）

```
name Send Bulk SMS
method GET
url https://api.textlocal.in/send/
body-mode json
seq 1

params
  1 apiKey secret
  0 numbers 998877665
/params

headers
  1 content-type application/json
/headers

body(type=json)
  {
    "key": "value"
  }
/body

script
  console.log('hello');
/script

tests
  expect(response.status).to.equal(200);
/tests
```

### 2.2 V2 格式（新版，推荐）

```
meta {
  name: Send Bulk SMS
  type: http
  seq: 1
  tags: [
    foo
    bar
  ]
}

get {
  url: https://api.textlocal.in/send/:id
  body: json
  auth: bearer
}

params:query {
  apiKey: secret
  numbers: 998877665
  "key with spaces": is allowed
  ~disabled-key: value
}

params:path {
  id: 123
}

headers {
  content-type: application/json
  Authorization: Bearer 123
}

body:json {
  {
    "hello": "world"
  }
}
```

#### 2.2.1 语法特性

| 特性 | 说明 |
|------|------|
| **块结构** | 使用 `tagname { ... }` 格式定义块 |
| **键值对** | `key: value` 格式，支持带引号的键名 |
| **禁用项** | 键名前加 `~` 表示禁用 |
| **列表** | `[ ... ]` 格式，用于 tags 等 |
| **多行文本** | 支持 JSON、XML、文本、GraphQL 等多种 body 格式 |
| **注释** | 暂不支持 |

#### 2.2.2 支持的块类型

```
# 元数据
meta { ... }

# HTTP 方法
get { ... }
post { ... }
put { ... }
delete { ... }
patch { ... }
options { ... }
head { ... }
connect { ... }
trace { ... }
http { ... }  # 自定义方法

# gRPC 和 WebSocket
grpc { ... }
ws { ... }

# 参数
params:query { ... }
params:path { ... }
query { ... }  # 别名

# 头信息
headers { ... }
metadata { ... }  # gRPC 元数据

# 认证
auth:basic { ... }
auth:bearer { ... }
auth:digest { ... }
auth:oauth1 { ... }
auth:oauth2 { ... }
auth:awsv4 { ... }
auth:wsse { ... }
auth:apikey { ... }
auth:ntlm { ... }

# 请求体
body:json { ... }
body:text { ... }
body:xml { ... }
body:sparql { ... }
body:graphql { ... }
body:graphql:vars { ... }
body:form-urlencoded { ... }
body:multipart-form { ... }
body:file { ... }

# 变量和断言
vars:pre-request { ... }
vars:post-response { ... }
assert { ... }

# 脚本和测试
script:pre-request { ... }
script:post-response { ... }
tests { ... }

# 文档
docs { ... }

# 示例
example { ... }

# 设置
settings { ... }
```

## 3. 解析器架构

### 3.1 技术栈

| 组件 | 说明 |
|------|------|
| **Ohm.js** | V2 解析器使用的 PEG（Parsing Expression Grammar）语法解析库 |
| **arcsecond** | V1 解析器使用的函数式解析器组合子库 |
| **Lodash** | 用于对象合并和数据转换 |

### 3.2 核心文件结构

```
packages/bruno-lang/
├── src/
│   └── index.js          # 入口文件，导出所有版本
├── v1/
│   ├── src/
│   │   ├── index.js          # V1 主解析器
│   │   ├── inline-tag.js     # 行内标签解析
│   │   ├── params-tag.js     # 参数标签解析
│   │   ├── headers-tag.js    # 头信息标签解析
│   │   ├── body-tag.js       # 请求体标签解析
│   │   ├── script-tag.js     # 脚本标签解析
│   │   ├── tests-tag.js      # 测试标签解析
│   │   ├── env-vars-tag.js   # 环境变量标签解析
│   │   └── utils.js          # 工具函数
│   └── tests/
└── v2/
    ├── src/
    │   ├── bruToJson.js      # V2 主解析器（Ohm.js 语法定义）
    │   ├── jsonToBru.js      # JSON 转 Bru
    │   ├── envToJson.js      # 环境文件解析
    │   ├── jsonToEnv.js      # JSON 转环境文件
    │   ├── dotenvToJson.js   # .env 文件解析
    │   ├── collectionBruToJson.js  # 集合文件解析
    │   ├── jsonToCollectionBru.js  # JSON 转集合文件
    │   ├── utils.js          # 工具函数
    │   └── example/          # 示例解析
    └── tests/
```

## 4. 解析步骤（V2 版本）

### 4.1 语法定义（Ohm Grammar）

解析器首先使用 Ohm.js 定义完整的 Bru 语法：

```javascript
const grammar = ohm.grammar(`Bru {
  BruFile = (meta | http | grpc | ws | query | params | headers | metadata | auths | bodies | varsandassert | script | tests | settings | docs | example)*
  
  # 字典块 - 键值对形式
  dictionary = st* "{" st* pairlist? tagend
  pairlist = optionalnl* pair (~tagend stnl* pair)* (~tagend space)*
  pair = st* pairannotations st* (quoted_key | key) st* ":" st* value st*
  
  # 文本块 - 自由文本形式
  textblock = textline (~tagend nl textline)*
  
  # 列表块 - 列表项形式
  list = st* "[" nl+ listitems? st* nl+ st* "]"
  
  # ... 更多语法规则
}`);
```

### 4.2 语义操作（Semantic Actions）

语法匹配成功后，通过语义属性（Semantics）将 AST 转换为 JSON：

```javascript
const sem = grammar.createSemantics().addAttribute('ast', {
  BruFile(tags) {
    // 合并所有标签的解析结果
    return _.reduce(tags.ast, (result, item) => {
      return _.mergeWith(result, item, concatArrays);
    }, {});
  },
  
  // 字典块处理
  dictionary(_1, _2, _3, pairlist, _4) {
    return pairlist.ast;
  },
  
  // 键值对处理
  pair(_1, annotations, _2, key, _3, _4, _5, value, _6) {
    let res = {};
    res[key.ast] = value.ast ? value.ast.trim() : '';
    // 处理注解
    const annotationList = annotations.ast;
    if (annotationList && annotationList.length > 0) {
      res[ANNOTATIONS_KEY] = annotationList;
    }
    return res;
  },
  
  // HTTP 方法处理
  get(_1, dictionary) {
    return {
      http: {
        method: 'get',
        ...mapPairListToKeyValPair(dictionary.ast)
      }
    };
  },
  
  // Headers 处理
  headers(_1, dictionary) {
    return {
      headers: mapPairListToKeyValPairs(dictionary.ast)
    };
  },
  
  // Body JSON 处理
  bodyjson(_1, _2, _3, _4, textblock, _5) {
    return {
      body: {
        json: outdentString(textblock.sourceString)
      }
    };
  }
});
```

### 4.3 解析流程

```
Bru 纯文本
    ↓
[Ohm Grammar 匹配]
    ↓
语法验证
    ↓
[Semantic Actions 转换]
    ↓
中间 AST（抽象语法树）
    ↓
[数据转换与合并]
    ↓
最终 JSON 结构
```

### 4.4 关键转换函数

#### 4.4.1 `mapPairListToKeyValPairs

将字典块的键值对列表转换为标准格式：

```javascript
const mapPairListToKeyValPairs = (pairList = [], parseEnabled = true) => {
  return _.map(pairList[0], (pair) => {
    let name = _.keys(pair)[0];
    let value = pair[name];
    
    // 解析启用/禁用状态
    let enabled = true;
    if (name && name.length && name.charAt(0) === '~') {
      name = name.slice(1);
      enabled = false;
    }
    
    return { name, value, enabled };
  });
};
```

#### 4.4.2 `outdentString`

移除文本块的缩进：

```javascript
const outdentString = (str, spaces = 2) => {
  const spacesRegex = new RegExp(`^ {${spaces}}`);
  return str
    .split(/\r\n|\r|\n/)
    .map((line) => line.replace(spacesRegex, ''))
    .join('\n');
};
```

## 5. 生成的中间结构（JSON）

### 5.1 HTTP 请求结构

```json
{
  "meta": {
    "name": "Send Bulk SMS",
    "type": "http",
    "seq": "1",
    "tags": ["foo", "bar"]
  },
  "http": {
    "method": "get",
    "url": "https://api.textlocal.in/send/:id",
    "body": "json",
    "auth": "bearer"
  },
  "params": [
    {
      "name": "apiKey",
      "value": "secret",
      "type": "query",
      "enabled": true
    },
    {
      "name": "id",
      "value": "123",
      "type": "path",
      "enabled": true
    }
  ],
  "headers": [
    {
      "name": "content-type",
      "value": "application/json",
      "enabled": true
    }
  ],
  "auth": {
    "bearer": {
      "token": "123"
    },
    "basic": {
      "username": "john",
      "password": "secret"
    }
  },
  "body": {
    "json": "{\n  \"hello\": \"world\"\n}",
    "text": "This is a text body",
    "formUrlEncoded": [
      { "name": "apikey", "value": "secret", "enabled": true }
    ],
    "multipartForm": [
      { "name": "file", "type": "file", "value": ["path/to/file"], "enabled": true }
    ]
  },
  "vars": {
    "req": [
      { "name": "departingDate", "value": "2020-01-01", "local": false, "enabled": true }
    ],
    "res": [
      { "name": "token", "value": "$res.body.token", "local": false, "enabled": true }
    ]
  },
  "assertions": [
    { "name": "$res.status", "value": "200", "enabled": true }
  ],
  "script": {
    "req": "const foo = 'bar';"
  },
  "tests": "function onResponse(request, response) {\n  expect(response.status).to.equal(200);\n}",
  "docs": "This request needs auth token to be set in the headers."
}
```

### 5.2 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| meta | Object | 请求元数据（名称、类型、序号、标签） |
| http | Object | HTTP 请求信息（方法、URL 等） |
| params | Array | 请求参数数组（query/path） |
| headers | Array | 请求头数组 |
| auth | Object | 认证配置（支持多种认证方式） |
| body | Object | 请求体（支持多种格式） |
| vars | Object | 预请求和响应后变量 |
| assertions | Array | 断言列表 |
| script | Object | 预请求和响应后脚本 |
| tests | String | 测试脚本 |
| docs | String | 文档描述 |

## 6. V1 与 V2 版本对比

### 6.1 主要差异

| 特性 | V1 | V2 |
|------|----|----|
| 语法风格 | 行内标签 + 开始/结束标签 | 块结构（类似 JSON） |
| 解析技术 | arcsecond（解析器组合子） | Ohm.js（PEG 语法） |
| 键值对格式 | 数字开头表示启用状态 | `~` 前缀表示禁用 |
| 支持的块类型 | 较少 | 丰富（支持 gRPC、WebSocket、多种认证） |
| 扩展性 | 有限 | 良好（可轻松添加新块类型） |

### 6.2 版本迁移

V2 版本完全向后兼容，推荐使用 V2 格式。

## 7. 关键技术点

### 7.1 Ohm.js 语法解析优势

1. **声明式语法**：使用 PEG 语法定义，可读性高
2. **分离的语义操作**：语法匹配与语义转换分离
3. **错误报告**：内置详细的语法错误定位
4. **可扩展性**：易于扩展新的语法规则

### 7.2 文本块处理策略

1. **缩进处理**：自动处理文本块的缩进
2. **多行文本**：支持使用 `'''` 包裹的多行文本
3. **内容类型注解**：支持 `@contentType(...)` 注解

### 7.3 键名处理

1. **特殊字符支持**：键名包含特殊字符时自动加引号
2. **转义字符**：支持 `\"` 转义引号
3. **禁用标记**：`~` 前缀标记禁用项

## 8. 使用示例

### 8.1 基本用法

```javascript
const { bruToJsonV2, jsonToBruV2 } = require('bruno-lang');

// Bru 转 JSON
const bruContent = fs.readFileSync('request.bru', 'utf8');
const json = bruToJsonV2(bruContent);

// JSON 转 Bru
const bru = jsonToBruV2(json);
```

### 8.2 环境文件解析

```javascript
const { bruToEnvJsonV2 } = require('bruno-lang');

const envContent = fs.readFileSync('Local.bru', 'utf8');
const envJson = bruToEnvJsonV2(envContent);
```

## 9. 扩展与维护

### 9.1 添加新的块类型

1. 在 Ohm grammar 中添加语法规则
2. 在语义操作中添加转换逻辑
3. 添加对应的测试用例

### 9.2 常见问题排查

1. **语法错误**：检查 Ohm grammar 定义是否正确
2. **转换错误**：检查语义操作中的映射逻辑
3. **缩进问题**：确认文本块的缩进处理
4. **特殊字符**：检查键名和值中的特殊字符处理

## 10. 总结

Bru 解析器通过 Ohm.js 提供了强大的语法解析能力，支持丰富的 HTTP 请求定义功能。从 V1 到 V2 的演进中，语法设计更加清晰、扩展性更强，为 Bruno API 客户端提供了坚实的基础。

主要特点：
- 基于 Ohm.js 的 PEG 语法解析
- 支持多种请求类型和认证方式
- 灵活的变量和脚本支持
- 完整的双向转换（Bru ↔ JSON）
- 良好的扩展性和可维护性
