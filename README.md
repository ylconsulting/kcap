# KCAP — Knowledge Capsule Format

> 面向知识库导入与 RAG 场景的轻量知识封装格式。

## What is KCAP?

KCAP（Knowledge Capsule）将异构来源（PDF、Word、网页、图片、扫描件等）解析后的知识内容，封装为一个可交换、可追溯的标准单元。

* **`knowledge.md`** — 主体知识文本（Markdown）
* **`kcap.json`** — 唯一清单入口（JSON）
* **`knowledge.map.json`** — 可选来源锚点映射
* **`resources/`** — 可选资源附件
* **`extras/`** — 可选扩展附属内容

KCAP 支持目录态（Loose Form）与 ZIP 打包态（Package Form）两种等价形态。

## Quick Start

最小 capsule 只需两个文件：

```text
report.kcap/
├── kcap.json
└── knowledge.md
```

**kcap.json**

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

**knowledge.md**

```markdown
# 2025年度行业报告

## 摘要

本文分析了 2025 年行业发展趋势。
```

## Specification

完整的 KCAP v1.0 正式规范请参阅 **[SPEC.md](SPEC.md)**。

## License

本规范以 [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/) 许可发布。
