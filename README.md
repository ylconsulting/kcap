# KCAP: Knowledge Capsule Format Version 1.0

## Draft

## Status of This Memo

本文档定义 **KCAP v1.0** 的格式与一致性要求。本文档中的 **MUST**、**SHOULD**、**MAY** 按 RFC 2119 / RFC 8174 的含义解释。

## Abstract

KCAP（Knowledge Capsule）是一种面向知识库导入与 RAG 场景的轻量知识封装格式。
KCAP 以 `knowledge.md` 作为主体知识文本，以 `kcap.json` 作为唯一清单入口，并可附带资源文件与扩展附属文件。KCAP 支持单源与受控多源封装，并支持目录态与打包态两种等价承载形态。

---

## 1. Introduction

异构资料（如 PDF、Word、网页、图片、扫描件）在进入知识库前，通常需要被解析、规整并统一表达。KCAP 的目标是为这一过程提供一个最小、稳定、可交换的封装格式。

KCAP v1.0 的设计目标如下：

* 以文本为核心表达知识内容
* 保留来源与关键元信息
* 支持必要的非文本资源附带
* 支持多个原始文件共同构成一个逻辑知识单元
* 保持格式轻量，不引入索引、向量或复杂关系图

---

## 2. Scope

KCAP v1.0 适用于：

* 单个原始文件的知识封装
* 多个原始文件共同构成一个逻辑知识单元的封装
* 作为切片、向量化、索引生成等下游流程的上游输入

KCAP v1.0 **不**定义：

* embedding、倒排索引或其他索引格式
* 复杂版本关系、依赖关系或文档图
* 任意文档集合的聚合封装

一个 KCAP capsule **MUST** 对应一个可独立消费的**逻辑知识单元**。若多个原始文件本身是多个独立知识单元，则 **SHOULD** 拆分为多个 capsule。

---

## 3. Terminology

**Capsule**
一个 KCAP 封装对象。

**Source**
capsule 的原始来源。一个 capsule **MAY** 有一个或多个 sources。

**Resource**
与知识内容直接相关、且可能在正文中被引用的附属资源。

**Extra**
非核心、可忽略的附属内容，用于扩展、调试、快照或平台特定用途。

**Loose Form**
目录形态的 KCAP。

**Package Form**
单文件形态的 KCAP。

---

## 4. Physical Forms

### 4.1 Loose Form

Loose Form 是一个目录，目录名 **SHOULD** 以 `.kcap` 结尾。

### 4.2 Package Form

Package Form 是一个单文件，扩展名 **MUST** 为 `.kcap`。
Package Form **MUST** 为 ZIP 容器，其解包后的内部结构 **MUST** 与 Loose Form 等价。

---

## 5. Package Layout

一个合法的 KCAP v1.0 capsule **MUST** 具有如下最小结构：

```text
example.kcap/
├── kcap.json
└── knowledge.md
```

以下目录为可选：

```text
resources/
extras/
```

完整结构如下：

```text
example.kcap/
├── kcap.json
├── knowledge.md
├── knowledge.map.json   ; optional
├── resources/           ; optional
└── extras/              ; optional
```

其中：

* `kcap.json` 是唯一清单文件
* `knowledge.md` 是主体知识文本
* `knowledge.map.json` 是 `knowledge.md` 的可选来源锚点映射文件
* `resources/` 保存知识相关资源
* `extras/` 保存非核心扩展附属内容

---

## 6. Core Files

### 6.1 `kcap.json`

`kcap.json` **MUST** 存在，且 **MUST** 为合法 JSON 对象。
它是 capsule 的唯一清单入口。

`kcap.json` **MUST** 包含以下字段：

* `version`
* `sources`

`version` 在本版本中 **MUST** 为：

```json
"1.0"
```

`id` **MAY** 存在，但 KCAP v1.0 **MUST NOT** 要求其具有全局唯一性。

### 6.2 `knowledge.md`

`knowledge.md` **MUST** 存在。
`knowledge.md` **MUST** 为 UTF-8 编码的 Markdown 文本。

`knowledge.md` **SHOULD** 以知识内容表达为优先，而非保留原始文档的视觉版式。实现方 **SHOULD** 保留标题层级、段落、列表、表格等语义结构。

若正文引用 `resources/` 中内容，**SHOULD** 使用相对路径，例如：

```markdown
![Figure](resources/fig-001.png)
```

实现方 **SHOULD** 在 `kcap.json` 中通过 `knowledge.content_hash` 记录 `knowledge.md` 的内容哈希（推荐 SHA-256），以支持入库时的去重校验与完整性验证。

### 6.3 `knowledge.map.json`

KCAP capsule **MAY** 包含一个名为 `knowledge.map.json` 的可选 sidecar 文件；若该文件存在，则其语义为 `knowledge.md` 的来源锚点映射文件。

`knowledge.map.json` 用于将 `knowledge.md` 中的块或片段映射回一个或多个原始来源位置，支持来源追溯、调试核查、高保真解释，以及为后续 chunk、citation、UI 高亮与定位跳转留接口。

消费者 **MAY** 忽略 `knowledge.map.json`，而不影响 capsule 的基本读取与使用。

`knowledge.map.json` 通过文件名约定发现，位于 capsule 根目录，与 `knowledge.md` 并列。不需要在 `kcap.json` 中显式配置，也不放入 `extras/`。

#### 6.3.1 数据模型

`knowledge.map.json` **MUST** 为合法 JSON 对象，包含以下顶层字段：

* `version`：source map 文件自身版本，独立于 KCAP 主版本管理。
* `target`：固定为 `"knowledge.md"`，标明此 map 对应的正文文件。
* `mappings`：映射记录数组。

#### 6.3.2 映射记录

每条映射记录描述 `knowledge.md` 中一个知识块到一个或多个原始来源位置的对应关系。

单条映射记录 **SHOULD** 包含以下字段：

* `anchor_id`：块级锚点 ID，用于稳定标识一条正文映射。
* `generated`：描述映射目标在 `knowledge.md` 中的位置。
  * `line_start`：起始行号
  * `line_end`：结束行号
  * 可选 `column_start`、`column_end`：列范围
  * 可选 `char_start`、`char_end`：字符范围
* `sources`：数组，表示该知识块对应的一个或多个来源位置（one-to-many）。

#### 6.3.3 来源锚点

`sources` 数组中每个元素 **SHOULD** 包含：

* `source_id`：对应 `kcap.json.sources[*].source_id`

以下定位字段为可选，根据来源类型按需使用：

* `page`：页码
* `bbox`：边界框坐标数组
* `char_range`：字符范围
* `offset_start`、`offset_end`：偏移量
* `resource_ref`：资源引用
* `confidence`：映射置信度

#### 6.3.4 设计边界

* v1.0 以**块级映射为主、位置级可选**，不采用纯行列级或压缩编码（如 VLQ）设计。
* v1.0 不强制要求在 `knowledge.md` 中写入显式 anchor 标记；映射通过 `knowledge.map.json` 中的块顺序和位置描述完成。

#### 6.3.5 示例

```json
{
  "version": "1.0",
  "target": "knowledge.md",
  "mappings": [
    {
      "anchor_id": "blk_001",
      "generated": {
        "line_start": 1,
        "line_end": 3
      },
      "sources": [
        {
          "source_id": "src_001",
          "page": 1,
          "bbox": [72, 120, 540, 220]
        }
      ]
    },
    {
      "anchor_id": "blk_002",
      "generated": {
        "line_start": 5,
        "line_end": 9
      },
      "sources": [
        {
          "source_id": "src_001",
          "page": 2,
          "bbox": [80, 160, 560, 420]
        },
        {
          "source_id": "src_002",
          "page": 1
        }
      ]
    }
  ]
}
```

---

## 7. Manifest Model

`kcap.json` 顶层对象 **MAY** 包含以下字段：

```json
{
  "version": "1.0",
  "id": "string",
  "title": "string",
  "summary": "string",
  "language": "string",
  "created_at": "string",
  "sources": [],
  "composition": {},
  "knowledge": {},
  "resources": {},
  "extras": {},
  "pipeline": {}
}
```

除 `version`、`sources` 外，其余字段均为可选。

顶层字段用于描述 **capsule 本身**；`knowledge` 用于描述 `knowledge.md` 所承载的**知识内容**。

### 7.1 Top-level Field Format Conventions

`id`
capsule 标识。实现方 **SHOULD** 使用 UUID v4 或 URI 以便在跨系统集成时进行去重与追溯。若实现方不保证全局唯一性，**SHOULD** 在文档中注明其唯一性范围。

`language`
capsule 的主要语言。**SHOULD** 使用 BCP 47 语言标签（例如 `"zh-CN"`、`"en-US"`）。

`created_at`
capsule 的创建时间。**MUST** 为 ISO 8601 格式的 UTC 时间字符串（例如 `"2026-03-11T10:00:00Z"`）。

---

## 8. Top-level vs `knowledge`

顶层描述类属性用于回答“**这是什么 capsule**”，例如：

* `version`
* `id`
* `title`
* `summary`
* `language`
* `created_at`
* `sources`
* `composition`
* `resources`
* `extras`
* `pipeline`

`knowledge` 用于回答“**这个 capsule 包含什么知识**”，例如：

* `keywords`
* `tags`
* `entities`
* `abstract`
* `knowledge_time`
* `topic`
* `sensitivity`
* `quality`

实现方 **SHOULD** 遵循以下原则：

* 若字段描述封装对象本身，则置于顶层
* 若字段描述正文知识内容的语义、适用性或质量，则置于 `knowledge`

---

## 9. Sources

### 9.1 General

`sources` **MUST** 为数组。即使仅有一个来源，也 **MUST** 使用数组表示。

每个 source 对象 **MUST** 包含：

* `source_id`
* `kind`
* `ref`

推荐结构如下：

```json
{
  "source_id": "src_001",
  "kind": "url",
  "ref": "https://example.com/report.pdf",
  "mime_type": "application/pdf",
  "filename": "report.pdf",
  "hash": "sha256:...",
  "role": "primary",
  "sequence": 1,
  "metadata": {},
  "snapshots": []
}
```

### 9.2 Source Fields

`source_id`
来源唯一标识，唯一性范围由生产方定义。

`kind`
来源类型。推荐值包括：`url`、`file`、`object`、`external_id`。

`ref`
来源引用。它可以是 URL、文件标识、对象存储键、外部系统 ID 等。

`role`
来源角色。推荐值包括：`primary`、`part`、`attachment`、`supplement`。

`sequence`
当多个来源存在顺序关系时使用。

`metadata`
来源级元信息，例如作者、创建时间、页数、原系统 ID、文件尺寸、EXIF、PDF 元信息等。

### 9.3 Snapshots

`snapshots` 为可选数组，用于描述封装时保留下来的来源快照。
其语义是“封装时保存了什么”，而不是“来源本身是什么”。

单个 snapshot 推荐结构如下：

```json
{
  "path": "extras/source.pdf",
  "kind": "source_snapshot",
  "mime_type": "application/pdf",
  "captured_at": "2026-03-11T10:00:00Z",
  "hash": "sha256:..."
}
```

`snapshot.path` **SHOULD** 指向 capsule 内部路径，通常位于 `extras/`。

---

## 10. Composition

当 capsule 仅包含单个来源且无特殊说明时，`composition` **MAY** 省略。

当 capsule 包含多个来源时，`composition` **SHOULD** 用于说明这些来源如何构成当前 capsule。

推荐结构如下：

```json
{
  "mode": "composed",
  "primary_source": "src_001",
  "ordering": "sequence"
}
```

`mode` 推荐值：

* `single`
* `composed`
* `merged`

其中：

* `single` 表示单来源
* `composed` 表示多个来源共同组成同一文档或对象
* `merged` 表示多个紧密相关来源经整理合并为一个知识单元

KCAP v1.0 的多源能力 **MUST NOT** 用于表达任意文档集合的打包。

---

## 11. Metadata Objects

### 11.1 `knowledge`

`knowledge` 用于保存知识内容的结构化信息，例如：

* `keywords`
* `tags`
* `entities`
* `abstract`
* `knowledge_time`
* `sensitivity`
* `quality`
* `content_hash`

该对象为开放对象。

#### 11.1.1 `content_hash`

`content_hash` **SHOULD** 记录 `knowledge.md` 文件内容的哈希值，格式 **SHOULD** 为 `"算法:十六进制摘要"`（例如 `"sha256:a1b2c3..."`）。入库系统 **MAY** 使用此字段进行去重检测与完整性校验。

#### 11.1.2 `quality`

`quality` 用于描述知识内容的质量评估信息，供入库流程进行质量门控。推荐结构如下：

```json
{
  "extraction_confidence": 0.95,
  "completeness": "full",
  "review_status": "auto",
  "issues": []
}
```

其中：

* `extraction_confidence`：内容提取置信度，取值 0.0–1.0。用于表示从原始来源提取为 Markdown 的可靠程度。
* `completeness`：内容完整性。推荐值：`"full"`、`"partial"`、`"degraded"`。
* `review_status`：审核状态。推荐值：`"auto"`（仅自动处理）、`"human_reviewed"`（经人工审核）、`"rejected"`（已拒绝）。
* `issues`：已知质量问题列表，每项为一个字符串，例如 `"OCR 识别部分文字模糊"`、`"表格结构丢失"`。

入库系统 **MAY** 依据 `quality` 字段决定是否接受该 capsule 入库。

### 11.2 `resources`

若存在 `resources/`，实现方 **MAY** 通过 `resources` 为其中的资源补充描述信息。
其键 **SHOULD** 为相对 `resources/` 的路径或文件名。

示例：

```json
{
  "fig-001.png": {
    "type": "image",
    "description": "市场趋势图"
  }
}
```

### 11.3 `extras`

若存在 `extras/`，实现方 **MAY** 通过 `extras` 字段描述其中的附加文件。

示例：

```json
{
  "source.pdf": {
    "path": "extras/source.pdf",
    "kind": "source_snapshot",
    "mime_type": "application/pdf"
  },
  "ocr-debug.json": {
    "path": "extras/ocr-debug.json",
    "kind": "debug_artifact",
    "mime_type": "application/json"
  }
}
```

标准消费者 **MAY** 忽略 `extras/` 及其描述，而不影响对 capsule 的基本读取。

### 11.4 `pipeline`

`pipeline` 用于记录 capsule 的生产流水线信息，使入库系统能够追溯 capsule 的生成过程、排查问题、并支持重新处理。

推荐结构如下：

```json
{
  "producer": "kcap-pipeline",
  "producer_version": "0.3.1",
  "created_at": "2026-03-11T10:00:00Z",
  "steps": [
    {
      "name": "pdf_extract",
      "tool": "pdfplumber",
      "tool_version": "0.10.0",
      "started_at": "2026-03-11T09:59:50Z",
      "finished_at": "2026-03-11T09:59:55Z",
      "params": {},
      "status": "success"
    },
    {
      "name": "markdown_convert",
      "tool": "custom_converter",
      "tool_version": "1.2.0",
      "started_at": "2026-03-11T09:59:55Z",
      "finished_at": "2026-03-11T09:59:58Z",
      "status": "success"
    }
  ]
}
```

其中：

* `producer`：生产 capsule 的系统名称。
* `producer_version`：生产系统版本。
* `created_at`：capsule 生产时间，**SHOULD** 为 ISO 8601 格式的 UTC 时间。
* `steps`：处理步骤数组，按执行顺序排列。每个步骤 **SHOULD** 包含 `name` 和 `status`。

入库系统 **MAY** 依据 `pipeline` 信息判断是否需要重新处理，或在排查问题时回溯生产流程。

---

## 12. Conformance

一个实现符合 KCAP v1.0，当且仅当其满足以下条件：

1. capsule **MUST** 包含 `kcap.json`
2. capsule **MUST** 包含 `knowledge.md`
3. `kcap.json` **MUST** 为合法 JSON
4. `kcap.json.version` **MUST** 为 `"1.0"`
5. `kcap.json.sources` **MUST** 存在且为数组
6. 每个 source **MUST** 包含 `source_id`、`kind`、`ref`
7. `knowledge.md` **MUST** 为 UTF-8 Markdown 文件
8. Package Form **MUST** 为 ZIP，且解包后结构与 Loose Form 等价
9. 若 `created_at` 存在，**MUST** 为 ISO 8601 格式的 UTC 时间字符串

### 12.1 Ingestion Validation Recommendations

入库系统在消费 KCAP capsule 时，**SHOULD** 额外执行以下校验：

* `knowledge.md` 非空且包含有效 Markdown 内容
* 若 `knowledge.content_hash` 存在，则验证其与 `knowledge.md` 实际内容一致
* 若 `knowledge.quality` 存在，检查 `extraction_confidence` 是否满足入库门槛
* 若 `resources/` 中存在文件，`knowledge.md` 中 **SHOULD** 存在对应的引用
* 若 `pipeline` 存在，验证所有步骤的 `status` 字段

上述校验为推荐实践，不属于合规性要求。

---

## 13. Minimal Example

### 13.1 Layout

```text
report.kcap/
├── kcap.json
└── knowledge.md
```

### 13.2 `kcap.json`

```json
{
  "version": "1.0",
  "title": "2025年度行业报告",
  "sources": [
    {
      "source_id": "src_001",
      "kind": "url",
      "ref": "https://example.com/report.pdf",
      "mime_type": "application/pdf",
      "role": "primary"
    }
  ]
}
```

### 13.3 `knowledge.md`

```markdown
# 2025年度行业报告

## 摘要

本文分析了 2025 年行业发展趋势。
```

---

## 14. Full Ingestion Example

以下示例展示入库系统处理一份 PDF 后生成的完整 capsule。

### 14.1 Layout

```text
report.kcap/
├── kcap.json
├── knowledge.md
├── knowledge.map.json
├── resources/
│   └── fig-001.png
└── extras/
    └── source.pdf
```

### 14.2 `kcap.json`

```json
{
  "version": "1.0",
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "title": "2025年度行业报告",
  "language": "zh-CN",
  "created_at": "2026-03-11T10:00:00Z",
  "sources": [
    {
      "source_id": "src_001",
      "kind": "url",
      "ref": "https://example.com/report.pdf",
      "mime_type": "application/pdf",
      "filename": "report.pdf",
      "hash": "sha256:e3b0c44298fc1c149afbf4c8996fb924...",
      "role": "primary",
      "metadata": {
        "page_count": 24,
        "author": "行业研究院"
      },
      "snapshots": [
        {
          "path": "extras/source.pdf",
          "kind": "source_snapshot",
          "mime_type": "application/pdf",
          "captured_at": "2026-03-11T09:59:50Z",
          "hash": "sha256:e3b0c44298fc1c149afbf4c8996fb924..."
        }
      ]
    }
  ],
  "knowledge": {
    "keywords": ["行业趋势", "2025", "市场分析"],
    "abstract": "本文分析了 2025 年行业发展趋势，涵盖市场规模、竞争格局与未来展望。",
    "content_hash": "sha256:a1b2c3d4e5f6...",
    "quality": {
      "extraction_confidence": 0.95,
      "completeness": "full",
      "review_status": "auto",
      "issues": []
    }
  },
  "resources": {
    "fig-001.png": {
      "type": "image",
      "description": "市场趋势图"
    }
  },
  "pipeline": {
    "producer": "kcap-pipeline",
    "producer_version": "0.3.1",
    "created_at": "2026-03-11T10:00:00Z",
    "steps": [
      {
        "name": "pdf_extract",
        "tool": "pdfplumber",
        "tool_version": "0.10.0",
        "started_at": "2026-03-11T09:59:50Z",
        "finished_at": "2026-03-11T09:59:55Z",
        "status": "success"
      },
      {
        "name": "markdown_convert",
        "tool": "custom_converter",
        "tool_version": "1.2.0",
        "started_at": "2026-03-11T09:59:55Z",
        "finished_at": "2026-03-11T09:59:58Z",
        "status": "success"
      }
    ]
  }
}
```

### 14.3 `knowledge.map.json`

```json
{
  "version": "1.0",
  "target": "knowledge.md",
  "mappings": [
    {
      "anchor_id": "blk_001",
      "generated": {
        "line_start": 1,
        "line_end": 3
      },
      "sources": [
        {
          "source_id": "src_001",
          "page": 1,
          "bbox": [72, 120, 540, 220]
        }
      ]
    },
    {
      "anchor_id": "blk_002",
      "generated": {
        "line_start": 5,
        "line_end": 12
      },
      "sources": [
        {
          "source_id": "src_001",
          "page": 2,
          "bbox": [80, 160, 560, 420]
        }
      ]
    }
  ]
}
```

---

## 15. Security and Integrity Considerations

实现方 **SHOULD** 为来源或快照记录内容哈希，以支持去重、校验与追溯。
消费方 **SHOULD** 将 `extras/` 中内容视为不可信输入。
若 capsule 来自外部系统，消费方 **SHOULD** 对 ZIP 解包、路径遍历、恶意文件名及超大文件做常规安全检查。

---

## 16. Extensibility

KCAP v1.0 采用“核心固定、外围开放”的扩展策略：

* 核心结构固定：`kcap.json`、`knowledge.md`、`sources`
* 标准可选 sidecar：`knowledge.map.json`
* 元信息对象开放：`sources[*].metadata`、`knowledge`、`resources`、`extras`、`pipeline`

私有扩展 **SHOULD NOT** 破坏核心字段语义。
通用消费者 **MUST NOT** 被要求理解私有扩展才能完成基本读取。

---

## 17. Future Work

后续版本可考虑增加但 v1.0 不包含：

* JSON Schema 定义，用于自动化校验 `kcap.json` 与 `knowledge.map.json`
* capsule 生命周期状态（draft → reviewed → approved → archived）
* chunk 标准
* embedding / index 标准
* 多语言正文
* 更细粒度的来源锚点映射（如正文内嵌稳定锚点标记、更复杂的映射规则）
* 权限与版权信息
* 扩展命名空间机制
* 入库批次（batch）描述协议
