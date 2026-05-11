# Bru 解析器文档最终审查报告

**审查日期**：2026-05-12
**审查对象**：`bru-parser.md`
**审查范围**：body.mode 映射表述一致性、全文表述准确性

---

## 一、修正内容汇总

本次审查完成以下修正，确保文档与源码行为 100% 一致：

### 1.1 字段映射表修正（第 687-688 行）

**修正前**：
| 源字段 | 目标字段 | 说明 |
|--------|---------|------|
| http/grpc/ws.body | request.body.mode | 当前选中的 body 模式 |

**修正后**：
| 源字段 | 目标字段 | 说明 |
|--------|---------|------|
| http.body | request.body.mode | 当前选中的 body 模式（仅 HTTP/GraphQL） |
| 硬编码默认值 | request.body.mode | gRPC/WS 的 body.mode 默认值 |

### 1.2 修正依据（源码验证）

```javascript
// 第 74-83 行：gRPC body 处理
transformedJson.request.body = _.get(json, 'body', {
  mode: 'grpc',  // ← 硬编码默认值，不读取 json.grpc.body
  grpc: _.get(json, 'body.grpc', [...])
});

// 第 84-94 行：WS body 处理
transformedJson.request.body = _.get(json, 'body', {
  mode: 'ws',    // ← 硬编码默认值，不读取 json.ws.body
  ws: _.get(json, 'body.ws', [...])
});

// 第 95-100 行：HTTP/GraphQL body 处理
transformedJson.request.body = _.get(json, 'body', {});
transformedJson.request.body.mode = _.get(json, 'http.body', 'none');
// ↑ 仅 HTTP/GraphQL 显式读取 http.body 设置 mode
```

---

## 二、全文一致性检查结果

### 2.1 三处关键表述已统一

| 位置 | 表述 | 状态 |
|------|------|------|
| 第 566-571 行（Body Mode 映射表） | gRPC/WS: 硬编码默认值，HTTP/GraphQL: json.http.body | ✅ 一致 |
| 第 573-576 行（注意提示） | 不来自 grpc.body/ws.body，仅 stringify 时写入 | ✅ 一致 |
| 第 687-688 行（完整字段映射表） | 区分 HTTP/GraphQL 与 gRPC/WS 的不同来源 | ✅ 一致 |
| 第 939-941 行（附录索引） | 明确硬编码在默认对象中，不读取上述字段 | ✅ 一致 |

### 2.2 已删除的旧表述

- ❌ ~~http/grpc/ws.body -> request.body.mode~~
- ❌ ~~来自 json.grpc.body / json.ws.body~~

---

## 三、潜在误解风险点（共 2 点）

### 风险点 1：gRPC/WS 有 body:* 块时可能缺失 mode 字段

**可能误解**：
> 即使有 body:json 等块，gRPC/WS 的 body 对象仍然会有 mode 字段 = 'grpc' 或 'ws'

**实际行为**：
当 Bru 文件包含 `body:json` 等块时：
```
_.get(json, 'body', { mode: 'grpc', ... })
          ↑
    json.body = { json: "..." } 存在，不回退到默认对象
```
最终 `body = { json: "..." }` → **无 mode 字段**！

**澄清说明**：
`_.get` 的默认参数仅在 `json.body` 为 `undefined` 时生效。若存在任何 `body:*` 块（如 body:json、body:text、body:grpc 等），直接使用该对象，不会合并默认 mode 字段。

**建议**：
执行层访问 `body.mode` 时需做存在性检查：
```javascript
const mode = request.body?.mode ?? (isGrpc ? 'grpc' : isWs ? 'ws' : 'none');
```

---

### 风险点 2：ws.request.method 走 http 分支逻辑可能引起混淆

**可能误解**：
> ws-request 的 method 字段应该读取 `ws.method` 或为空，不应走 http 分支

**实际行为**：
```javascript
method:
  requestType === 'grpc-request'
    ? _.get(json, 'grpc.method', '')
    : String(_.get(json, 'http.method') ?? '').toUpperCase()
// ↑ ws-request 走 else 分支，读取 json.http.method
```
纯 WS 请求通常无 http 块 → `http.method` = `undefined` → method = 空字符串 `''`

**澄清说明**：
这是**统一结构设计**的副作用，而非 bug：
1. 所有请求类型共享统一 request 对象结构（便于后续处理）
2. method 字段无条件设置（避免 "cannot read property 'method' of undefined"）
3. WS/gRPC 场景下 method 字段实际无业务意义（空字符串是合理默认）

**建议**：
WS/gRPC 请求处理逻辑中应忽略 method 字段，或文档明确标注：
> ws-request / grpc-request 的 request.method 仅为结构统一目的，无业务意义，值为空字符串。

---

## 四、最终审查结论

### 4.1 文档状态

| 检查项 | 结果 |
|--------|------|
| body.mode 映射表述 | ✅ 已修正，与源码一致 |
| 正文、映射表、附录三处表述 | ✅ 统一 |
| 旧表述残留 | ✅ 已清理 |

### 4.2 推荐阅读顺序

1. 先读 `bru-parser.md` 第 6 章（Filestore 层归一化）获取整体视图
2. 遇关键行为疑问时参考 `bru-parser-analysis.md` 的深度源码分析
3. 开发执行层代码时注意本报告提到的 2 个潜在误解风险点

### 4.3 后续维护建议

每次代码改动后，重新运行最小可复现示例验证：
- 纯 HTTP 无 method → method = ''
- 纯 WS 请求 → method = ''，body.mode = 'ws'
- gRPC + body:json → body.mode 缺失
