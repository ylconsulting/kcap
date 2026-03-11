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
├── resources/   ; optional
└── extras/      ; optional
```

其中：

* `kcap.json` 是唯一清单文件
* `knowledge.md` 是主体知识文本
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
  "knowledge_metadata": {},
  "resources_metadata": {},
  "extras": {},
  "pipeline": {}
}
```

除 `version`、`sources` 外，其余字段均为可选。

顶层字段用于描述 **capsule 本身**；`knowledge_metadata` 用于描述 `knowledge.md` 所承载的**知识内容**。

---

## 8. Top-level vs Knowledge Metadata

顶层描述类属性用于回答“**这是什么 capsule**”，例如：

* `version`
* `id`
* `title`
* `summary`
* `language`
* `created_at`
* `sources`
* `composition`
* `resources_metadata`
* `extras`
* `pipeline`

`knowledge_metadata` 用于回答“**这个 capsule 包含什么知识**”，例如：

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
* 若字段描述正文知识内容的语义、适用性或质量，则置于 `knowledge_metadata`

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

### 11.1 `knowledge_metadata`

`knowledge_metadata` 用于保存知识内容的结构化信息，例如：

* `keywords`
* `tags`
* `entities`
* `abstract`
* `knowledge_time`
* `sensitivity`
* `quality`

该对象为开放对象。

### 11.2 `resources_metadata`

若存在 `resources/`，实现方 **MAY** 通过 `resources_metadata` 为其中的资源补充描述信息。
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

## 14. Security and Integrity Considerations

实现方 **SHOULD** 为来源或快照记录内容哈希，以支持去重、校验与追溯。
消费方 **SHOULD** 将 `extras/` 中内容视为不可信输入。
若 capsule 来自外部系统，消费方 **SHOULD** 对 ZIP 解包、路径遍历、恶意文件名及超大文件做常规安全检查。

---

## 15. Extensibility

KCAP v1.0 采用“核心固定、外围开放”的扩展策略：

* 核心结构固定：`kcap.json`、`knowledge.md`、`sources`
* 元信息对象开放：`sources[*].metadata`、`knowledge_metadata`、`resources_metadata`、`extras`、`pipeline`

私有扩展 **SHOULD NOT** 破坏核心字段语义。
通用消费者 **MUST NOT** 被要求理解私有扩展才能完成基本读取。

---

## 16. Future Work

后续版本可考虑增加但 v1.0 不包含：

* chunk 标准
* embedding / index 标准
* 多语言正文
* 来源锚点映射
* 权限与版权信息
* 扩展命名空间机制
