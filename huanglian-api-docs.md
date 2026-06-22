# 黄连智能分析系统 - API 接口文档

**Base URL**: `https://huanglian.xyz/api`  
**版本**: v0.1.0  
**最后更新**: 2026-06-17

---

## 目录

- [通用说明](#通用说明)
- [健康检查](#1-健康检查)
- [聊天问答](#2-聊天问答)
  - [完整响应](#21-完整响应)
  - [流式输出](#22-流式输出sse)
- [图片上传](#3-图片上传)
- [质量评估](#4-质量评估)
- [产地溯源](#5-产地溯源)
- [报告生成](#6-报告生成)
- [历史记录](#7-历史记录)
  - [会话管理](#71-会话管理)
  - [消息管理](#72-消息管理)
- [错误码说明](#错误码说明)

---

## 通用说明

### 请求格式

- Content-Type: `application/json`
- 图片上传使用 `multipart/form-data`

### 统一响应格式

所有接口返回统一格式（部分接口除外）：

```json
{
  "code": 0,
  "message": "success",
  "data": { ... }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `code` | int | 业务状态码，`0` 表示成功 |
| `message` | string | 返回消息 |
| `data` | object | 返回数据 |

---

## 1. 健康检查

### `GET /api/health`

检查后端服务运行状态。

**请求**: 无参数

**响应**:

```json
{
  "data": {
    "status": "ok"
  }
}
```

**示例**:

```bash
curl https://huanglian.xyz/api/health
```

---

## 2. 聊天问答

### 2.1 完整响应

#### `POST /api/chat`

发送用户问题，等待完整分析结果后一次性返回。

**请求参数**:

```json
{
  "query": "黄连有哪些功效？",
  "include_trace": false,
  "session_context": {
    "session_id": "abc123",
    "current_sample_id": "HL-CQ-SZ-HS-LS-001",
    "current_image_id": "img_xxx",
    "latest_report_id": "report_xxx"
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `query` | string | ✅ | 用户输入问题 |
| `include_trace` | bool | ❌ | 是否返回工具调用轨迹，默认 `false` |
| `session_context` | object | ❌ | 会话上下文 |
| `session_context.session_id` | string | ❌ | 会话 ID（用于自动保存聊天记录） |
| `session_context.current_sample_id` | string | ❌ | 当前样本编号 |
| `session_context.current_image_id` | string | ❌ | 当前图片编号 |
| `session_context.latest_report_id` | string | ❌ | 最近报告编号 |

**响应**:

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "answer": "黄连的主要功效包括清热燥湿、泻火解毒...",
    "route": "rag_summarize",
    "route_label": "RAG 知识问答",
    "route_reason": "RAG 知识问答",
    "sample_id": "HL-CQ-SZ-HS-LS-001",
    "image_id": null,
    "report_id": null,
    "tool_count": 1,
    "tool_trace": [
      {
        "step": 1,
        "tool_name": "rag_summarize",
        "tool_args": {"query": "黄连有哪些功效？"},
        "status": "success",
        "via": "agent",
        "output_preview": "...",
        "error_message": null
      }
    ],
    "session_context": {},
    "resolved_context": {},
    "structured_result": { ... },
    "report_payload": null,
    "answer_card": { ... },
    "display_sections": [ ... ]
  }
}
```

**响应字段说明**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `answer` | string | Agent 回答文本 |
| `route` | string | 本轮路由：`rag_summarize` / `get_sample_profile` / `analyze_quality` / `analyze_traceability` / `generate_structured_report` / `missing_context` |
| `route_label` | string | 路由中文标签 |
| `route_reason` | string | 命中路由原因说明 |
| `sample_id` | string/null | 解析出的样本编号 |
| `image_id` | string/null | 解析出的图片编号 |
| `report_id` | string/null | 解析出的报告编号 |
| `tool_count` | int | 工具调用次数 |
| `tool_trace` | array | 工具调用轨迹（`include_trace=true` 时有值） |
| `structured_result` | object/null | 结构化分析结果 |
| `report_payload` | object/null | 报告展示载荷 |
| `answer_card` | object/null | 回答摘要卡片 |
| `display_sections` | array | 分段展示块 |

**示例**:

```bash
curl -X POST https://huanglian.xyz/api/chat \
  -H "Content-Type: application/json" \
  -d '{
    "query": "分析样本 HL-CQ-SZ-HS-LS-001 的质量",
    "include_trace": true,
    "session_context": {
      "current_sample_id": "HL-CQ-SZ-HS-LS-001"
    }
  }'
```

---

### 2.2 流式输出（SSE）

#### `POST /api/chat/stream`

发送用户问题，以 Server-Sent Events 方式实时返回回答内容。

**请求参数**: 与 `POST /api/chat` 相同。

**响应格式**: `text/event-stream`

流式返回多条 `data:` 消息：

```
data: {"type": "start", "query": "什么是黄连？"}

data: {"type": "chunk", "content": "一"}

data: {"type": "chunk", "content": "、结论"}

data: {"type": "chunk", "content": "\n黄连是中药材..."}

data: {"type": "end", "answer": "完整回答内容..."}
```

**事件类型**:

| type | 字段 | 说明 |
|------|------|------|
| `start` | `query` | 流式开始，返回原始问题 |
| `chunk` | `content` | 文本片段（逐字/逐句） |
| `end` | `answer` | 流式结束，返回完整回答 |
| `error` | `message` | 发生错误 |
| `warning` | `message` | 非致命警告（如会话不存在） |

**注意**：
- 流式接口目前只支持 RAG 知识问答，不走完整 Agent 图
- 不会返回 `route` / `tool_trace` / `structured_result` 等字段
- 如果 `session_context` 包含 `session_id`，消息会自动保存到数据库

**示例**:

```bash
curl -X POST https://huanglian.xyz/api/chat/stream \
  -H "Content-Type: application/json" \
  -d '{"query": "什么是味连、雅连、云连？"}'
```

---

## 3. 图片上传

#### `POST /api/upload/image`

上传黄连样本图片，用于辅助质量评估和产地溯源。

**请求格式**: `multipart/form-data`

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `sample_id` | string | ✅ | 关联样本 ID |
| `image_type` | string | ✅ | 图片类型（见下表） |
| `file` | file | ✅ | 图片文件 |

**图片类型**:

| 值 | 说明 |
|------|------|
| `original` | 原图 |
| `cross_section` | 断面图 |
| `powder` | 粉末图 |
| `package` | 包装图 |
| `other` | 其他 |

**响应**:

```json
{
  "image_id": "img_20260617_a1b2c3d4",
  "sample_id": "HL-CQ-SZ-HS-LS-001",
  "image_type": "original",
  "original_filename": "huanglian_01.jpg",
  "content_type": "image/jpeg",
  "file_size": 245678,
  "file_path": "data/huanglian/uploads/HL-CQ-SZ-HS-LS-001/original/huanglian_01.jpg",
  "processed_file_path": null,
  "width": 1024,
  "height": 768,
  "sha256": "abc123...",
  "created_at": "2026-06-17T08:00:00",
  "warnings": []
}
```

**示例**:

```bash
curl -X POST https://huanglian.xyz/api/upload/image \
  -F "sample_id=HL-CQ-SZ-HS-LS-001" \
  -F "image_type=original" \
  -F "file=@/path/to/image.jpg"
```

---

## 4. 质量评估

#### `POST /api/quality/predict`

对黄连样本进行质量评估分析。

**请求参数**:

```json
{
  "sample_id": "HL-CQ-SZ-HS-LS-001",
  "image_id": "img_xxx",
  "top_k": 3
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `sample_id` | string | ✅ | 样本 ID |
| `image_id` | string | ❌ | 上传接口返回的 image_id |
| `top_k` | int | ❌ | 相似图召回数量（1-10），默认 3 |

**响应**:

```json
{
  "sample_id": "HL-CQ-SZ-HS-LS-001",
  "task": "quality",
  "quality_label": "high",
  "confidence": 0.85,
  "evidence": [
    "小檗碱含量 6.8%，高于药典标准 5.5%",
    "外观形态完整，色泽正常"
  ],
  "retrieved_contexts": [
    "根据药典标准，黄连小檗碱含量不得低于 5.5%..."
  ],
  "retrieval_count": 4,
  "warnings": [],
  "suggestions": ["建议补充显微鉴定数据"],
  "image_id": "img_xxx",
  "image_evidence_used": true,
  "similar_image_count": 3,
  "similar_images": [
    {
      "rank": 1,
      "similarity": 0.92,
      "reference_image_id": "ref_001",
      "sample_id": "HL-REF-001",
      "image_type": "original",
      "quality_label": "high",
      "origin_label": "重庆石柱",
      "processed_file_path": "..."
    }
  ],
  "structured_summary": "样本小檗碱含量 6.8%，形态完整..."
}
```

**quality_label 取值**:

| 值 | 说明 |
|------|------|
| `high` | 高质量 |
| `medium` | 中等质量 |
| `low` | 低质量 |
| `unknown` | 无法判断 |

**示例**:

```bash
curl -X POST https://huanglian.xyz/api/quality/predict \
  -H "Content-Type: application/json" \
  -d '{
    "sample_id": "HL-CQ-SZ-HS-LS-001",
    "top_k": 3
  }'
```

---

## 5. 产地溯源

#### `POST /api/traceability/predict`

对黄连样本进行产地溯源分析。

**请求参数**:

```json
{
  "sample_id": "HL-CQ-SZ-HS-LS-001",
  "image_id": "img_xxx",
  "top_k": 3
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `sample_id` | string | ✅ | 样本 ID |
| `image_id` | string | ❌ | 上传接口返回的 image_id |
| `top_k` | int | ❌ | 相似图召回数量（1-10），默认 3 |

**响应**:

```json
{
  "sample_id": "HL-CQ-SZ-HS-LS-001",
  "task": "traceability",
  "predicted_origin": "重庆石柱",
  "confidence": 0.78,
  "evidence": [
    "ecology 数据显示海拔 1200m，与石柱产区吻合",
    "图片相似检索显示与石柱参考图相似度 0.85"
  ],
  "retrieved_contexts": [],
  "retrieval_count": 3,
  "warnings": ["产地溯源为辅助分析结果，不作为鉴定依据"],
  "suggestions": ["建议补充 GAP 种植记录"],
  "image_id": "img_xxx",
  "image_evidence_used": true,
  "similar_image_count": 3,
  "similar_images": [ ... ],
  "origin_votes": [
    {"origin_label": "重庆石柱", "score": 0.62, "support_count": 3},
    {"origin_label": "湖北恩施", "score": 0.24, "support_count": 2},
    {"origin_label": "四川峨眉", "score": 0.14, "support_count": 1}
  ],
  "structured_summary": "..."
}
```

**示例**:

```bash
curl -X POST https://huanglian.xyz/api/traceability/predict \
  -H "Content-Type: application/json" \
  -d '{
    "sample_id": "HL-CQ-SZ-HS-LS-001",
    "top_k": 5
  }'
```

---

## 6. 报告生成

### 6.1 生成报告

#### `POST /api/report/generate`

为指定样本生成综合分析报告（质量评估 + 产地溯源）。

**请求参数**:

```json
{
  "sample_id": "HL-CQ-SZ-HS-LS-001",
  "image_id": "img_xxx",
  "top_k": 3,
  "persist": true,
  "output_dir": "data/huanglian/reports/generated"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `sample_id` | string | ✅ | **正式报告必须提供** |
| `image_id` | string | ❌ | 用于增强报告证据 |
| `top_k` | int | ❌ | 相似图召回数量（1-10），默认 3 |
| `persist` | bool | ❌ | 是否持久化保存到磁盘，默认 `true` |
| `output_dir` | string | ❌ | 报告输出目录 |

**响应**:

```json
{
  "report_id": "report_20260617_a1b2c3d4",
  "report_type": "quality_traceability",
  "sample_id": "HL-CQ-SZ-HS-LS-001",
  "image_id": "img_xxx",
  "generated_at": "2026-06-17T08:30:00",
  "summary": {
    "overall_conclusion": "样本质量评估为 high，预测产地为重庆石柱...",
    "risk_level": "low",
    "next_actions": ["补充显微鉴定数据", "确认 GAP 种植记录"]
  },
  "quality_result": {
    "label": "high",
    "confidence": 0.85,
    "summary": "...",
    "evidence": [ ... ],
    "raw": { ... }
  },
  "traceability_result": {
    "origin_guess": "重庆石柱",
    "confidence": 0.78,
    "summary": "...",
    "evidence": [ ... ],
    "raw": { ... }
  },
  "evidence": [
    {
      "evidence_id": "ev_001",
      "source_type": "structured",
      "title": "小檗碱含量",
      "content": "6.8%，高于药典标准",
      "source": "chemistry.csv",
      "score": null,
      "confidence": 0.95,
      "metadata": {}
    }
  ],
  "meta": {},
  "output_files": {
    "json": "data/huanglian/reports/generated/report_20260617_a1b2c3d4.json",
    "markdown": "data/huanglian/reports/generated/report_20260617_a1b2c3d4.md"
  },
  "markdown_content": "# 黄连综合分析报告\n\n## 一、样本信息..."
}
```

**⚠️ 注意**: `sample_id` 为必填项，仅凭 `image_id` 不能生成正式报告。

**示例**:

```bash
curl -X POST https://huanglian.xyz/api/report/generate \
  -H "Content-Type: application/json" \
  -d '{"sample_id": "HL-CQ-SZ-HS-LS-001"}'
```

---

### 6.2 读取报告

#### `GET /api/report/detail/{report_id}`

读取已保存到磁盘的报告详情。

**路径参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| `report_id` | string | 报告 ID |

**查询参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| `output_dir` | string | 报告目录，默认 `data/huanglian/reports/generated` |

**响应**: 与 `POST /api/report/generate` 相同结构。

**示例**:

```bash
curl https://huanglian.xyz/api/report/detail/report_20260617_a1b2c3d4
```

---

## 7. 历史记录

### 7.1 会话管理

#### 7.1.1 创建会话

##### `POST /api/history/sessions`

```json
// 请求
{ "title": "新对话" }

// 响应
{
  "code": 0,
  "message": "success",
  "data": {
    "id": "a1b2c3d4",
    "title": "新对话",
    "created_at": "2026-06-17T08:00:00",
    "updated_at": "2026-06-17T08:00:00",
    "current_sample_id": null,
    "current_image_id": null,
    "current_image_type": "original",
    "latest_report_id": null
  }
}
```

#### 7.1.2 列出会话

##### `GET /api/history/sessions`

```json
{
  "code": 0,
  "data": [
    {
      "id": "a1b2c3d4",
      "title": "新对话",
      "created_at": "2026-06-17T08:00:00",
      "updated_at": "2026-06-17T08:30:00",
      "current_sample_id": "HL-CQ-SZ-HS-LS-001",
      "current_image_id": null,
      "current_image_type": "original",
      "latest_report_id": null
    }
  ]
}
```

#### 7.1.3 获取会话详情

##### `GET /api/history/sessions/{session_id}`

```json
{
  "code": 0,
  "data": {
    "id": "a1b2c3d4",
    "title": "新对话",
    "created_at": "2026-06-17T08:00:00",
    "updated_at": "2026-06-17T08:30:00",
    "current_sample_id": "HL-CQ-SZ-HS-LS-001",
    "current_image_id": "img_xxx",
    "current_image_type": "original",
    "latest_report_id": "report_xxx"
  }
}
```

#### 7.1.4 更新会话

##### `PUT /api/history/sessions/{session_id}`

```json
// 请求（只传需要更新的字段）
{
  "title": "更新后的标题",
  "current_sample_id": "HL-NEW-001"
}

// 响应
{ "code": 0, "message": "success" }
```

#### 7.1.5 删除会话

##### `DELETE /api/history/sessions/{session_id}`

级联删除会话及其所有消息。

```json
{ "code": 0, "message": "success" }
```

---

### 7.2 消息管理

#### 7.2.1 获取消息

##### `GET /api/history/sessions/{session_id}/messages`

```json
{
  "code": 0,
  "data": [
    {
      "id": 1,
      "session_id": "a1b2c3d4",
      "role": "user",
      "content": "黄连有哪些功效？",
      "created_at": "2026-06-17T08:00:00",
      "route": null,
      "tool_trace": null,
      "structured_result": null,
      "report_payload": null
    },
    {
      "id": 2,
      "session_id": "a1b2c3d4",
      "role": "assistant",
      "content": "黄连的主要功效包括...",
      "created_at": "2026-06-17T08:00:05",
      "route": "rag_summarize",
      "tool_trace": [ ... ],
      "structured_result": { ... },
      "report_payload": null
    }
  ]
}
```

#### 7.2.2 添加消息

##### `POST /api/history/messages`

```json
// 请求
{
  "session_id": "a1b2c3d4",
  "role": "user",
  "content": "请分析这个样本的质量",
  "route": null,
  "tool_trace": null,
  "structured_result": null,
  "report_payload": null
}

// 响应
{
  "code": 0,
  "data": { "message_id": 3 }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `session_id` | string | ✅ | 会话 ID |
| `role` | string | ✅ | `user` 或 `assistant` |
| `content` | string | ✅ | 消息内容 |
| `route` | string | ❌ | 路由名称 |
| `tool_trace` | array | ❌ | 工具调用轨迹 |
| `structured_result` | object | ❌ | 结构化结果 |
| `report_payload` | object | ❌ | 报告载荷 |

---

## 错误码说明

### HTTP 状态码

| 状态码 | 说明 |
|------|------|
| `200` | 请求成功 |
| `400` | 请求参数错误 |
| `404` | 资源不存在 |
| `405` | 请求方法不允许（如 GET 请求到 POST 接口） |
| `500` | 服务器内部错误 |

### 业务状态码

| code | 说明 |
|------|------|
| `0` | 成功 |

### 常见错误示例

**参数错误 (400)**:
```json
{ "detail": "质量评估失败: sample_id 不能为空" }
```

**资源不存在 (404)**:
```json
{ "detail": "报告不存在: report_xxx" }
```

**服务器错误 (500)**:
```json
{ "detail": "chat接口调用失败: 模型调用失败..." }
```

---

## 附录：接口速查表

| 接口 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 健康检查 | GET | `/api/health` | 服务状态检查 |
| 聊天问答 | POST | `/api/chat` | 完整响应 |
| 流式聊天 | POST | `/api/chat/stream` | SSE 流式输出 |
| 图片上传 | POST | `/api/upload/image` | 上传样本图片 |
| 质量评估 | POST | `/api/quality/predict` | 样本质量评估 |
| 产地溯源 | POST | `/api/traceability/predict` | 样本产地溯源 |
| 生成报告 | POST | `/api/report/generate` | 生成综合报告 |
| 报告详情 | GET | `/api/report/detail/{report_id}` | 读取已保存报告 |
| 创建会话 | POST | `/api/history/sessions` | 新建聊天会话 |
| 列出会话 | GET | `/api/history/sessions` | 获取所有会话 |
| 会话详情 | GET | `/api/history/sessions/{id}` | 获取会话详情 |
| 更新会话 | PUT | `/api/history/sessions/{id}` | 更新会话信息 |
| 删除会话 | DELETE | `/api/history/sessions/{id}` | 删除会话（级联） |
| 获取消息 | GET | `/api/history/sessions/{id}/messages` | 获取会话消息 |
| 添加消息 | POST | `/api/history/messages` | 手动添加消息 |

---

**免责声明**: 本系统输出仅作为黄连样本分析辅助结果，不替代药典检验、专业鉴定、医疗建议或商业质检结论。
