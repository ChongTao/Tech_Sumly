# 3 SQLite FTS5 全文检索

SQLite FTS5（Full-Text Search 5）是 SQLite 内置的全文检索扩展。它为文本维护倒排索引，支持 `MATCH` 查询、短语与前缀匹配、结果高亮和 BM25 排序，无需额外部署搜索服务。

FTS5 适合将搜索能力嵌入桌面应用、移动端、本地知识库、轻量服务和个人记忆系统等场景。它把业务数据与索引存放在同一个 SQLite 文件中，部署简单、离线可用；对于分布式索引、极高并发或复杂分析需求，则应评估 OpenSearch、Elasticsearch 等专用搜索引擎。

## 1 核心概念

- **虚表**：通过 `CREATE VIRTUAL TABLE ... USING fts5(...)` 创建，FTS5 会为指定列维护全文索引。
- **倒排索引**：按词项记录其出现的文档，避免逐篇扫描正文。
- **Tokenizer**：决定如何把文本切分为可搜索的词项，直接影响召回质量。
- **MATCH**：FTS5 的全文查询运算符；查询语句不是普通的 SQL `LIKE`。
- **BM25**：内置相关性排序函数。`bm25()` 返回值越小，结果越相关。

## 2 基本用法

### 2.1 快速体验

如果暂时不需要保存额外业务字段，可以直接创建 FTS5 虚表、写入文本并查询：

```sql
CREATE VIRTUAL TABLE notes_fts USING fts5(title, body);

INSERT INTO notes_fts(title, body) VALUES
  ('SQLite FTS5', 'FTS5 使用倒排索引提供全文搜索'),
  ('个人记忆', '可索引笔记、会话摘要和网页剪藏');

SELECT rowid, title, body
FROM notes_fts
WHERE notes_fts MATCH '全文 搜索';
```

虚表中的 `rowid` 是文档标识。上述方式适合快速原型；当需要时间、标签、权限等业务字段时，使用普通表保存原始数据，再为其建立 FTS5 索引。

### 2.2 为已有数据表建立索引

推荐使用外部内容表模式：业务字段保存在普通表中，FTS5 虚表只保存索引。

```sql
CREATE TABLE documents (
  id INTEGER PRIMARY KEY,
  title TEXT NOT NULL,
  content TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE VIRTUAL TABLE documents_fts USING fts5(
  title,
  content,
  content='documents',
  content_rowid='id',
  tokenize='unicode61'
);
```

为已有数据首次构建索引：

```sql
INSERT INTO documents_fts(documents_fts) VALUES ('rebuild');
```

外部内容表的增删改不会自动同步至 FTS5 索引。应通过触发器、统一的应用层写入流程或定期重建，确保 `documents` 与 `documents_fts` 一致。

例如，使用触发器同步新增、更新和删除：

```sql
CREATE TRIGGER documents_ai AFTER INSERT ON documents BEGIN
  INSERT INTO documents_fts(rowid, title, content)
  VALUES (new.id, new.title, new.content);
END;

CREATE TRIGGER documents_ad AFTER DELETE ON documents BEGIN
  INSERT INTO documents_fts(documents_fts, rowid, title, content)
  VALUES ('delete', old.id, old.title, old.content);
END;

CREATE TRIGGER documents_au AFTER UPDATE ON documents BEGIN
  INSERT INTO documents_fts(documents_fts, rowid, title, content)
  VALUES ('delete', old.id, old.title, old.content);
  INSERT INTO documents_fts(rowid, title, content)
  VALUES (new.id, new.title, new.content);
END;
```

## 3 常见查询方式

FTS5 的查询入口是 `MATCH`。查询表达式与 SQL 参数分开：`MATCH :query` 中的 `:query` 应由参数绑定提供，而不是拼接到 SQL 字符串中。

| 写法 | 含义 |
| --- | --- |
| `向量 检索` | 同时包含两个词，空格等价于隐式 AND。 |
| `BM25 OR FTS5` | 至少包含一个词；布尔运算符应使用大写。 |
| `RAG NOT 测试` | 包含 `RAG` 且不包含 `测试`。 |
| `"混合 检索"` | 按相邻词项进行短语匹配。 |
| `retriev*` | 前缀匹配，例如 `retrieve`、`retrieval`。 |
| `title : FTS5` | 仅在 `title` 列中匹配。 |
| `NEAR(向量 检索, 5)` | 两个词项相距不超过 5 个词项。 |

查询语法错误（例如未闭合的引号）会导致 SQLite 返回错误。面向终端用户的搜索框应捕获该错误；若产品只需要简单关键词搜索，也可以在应用层转义或限制可使用的操作符。

## 4 相关性排序与结果高亮

`MATCH` 只负责筛选命中文档，并不保证返回顺序就是最相关的顺序。使用 `bm25()` 明确排序；它的结果通常为负数，**数值越小，相关性越高**：

```sql
SELECT
  d.id,
  d.title,
  snippet(documents_fts, 1, '<mark>', '</mark>', '…', 16) AS excerpt,
  bm25(documents_fts, 5.0, 1.0) AS score
FROM documents_fts
JOIN documents AS d ON d.id = documents_fts.rowid
WHERE documents_fts MATCH :query
ORDER BY score
LIMIT 10;
```

示例中 `5.0, 1.0` 分别是标题和正文的权重，因此标题命中的影响更大。权重的顺序必须与 FTS5 表的列顺序一致；应根据真实查询集调参，而不是只凭直觉增大权重。

当相关性接近时，可增加业务规则作为稳定的次级排序，例如优先近期文档：

```sql
ORDER BY bm25(documents_fts, 5.0, 1.0), d.updated_at DESC
```

`snippet()` 返回包含命中词高亮的摘要：第二个参数为列索引，`1` 表示 `content`；`<mark>` 与 `</mark>` 是前后标记；最后的 `16` 是摘要 token 数。展示到 HTML 页面时，仍需按页面渲染规则处理原始文本，避免将文档内容直接作为可信 HTML。

## 5 分词与中文检索

`unicode61` 适合一般 Unicode 文本，但不是中文分词器。中文语料需要使用真实查询集验证召回效果；必要时应在入库和查询时应用同一套中文分词预处理，或接入 ICU / 自定义 tokenizer。

对文件名、路径、错误码等需要子串匹配的字段，可以单独评估 FTS5 的 `trigram` tokenizer；它更适合较长子串，短查询词的效果需要实测。不同字段可以按检索意图采用不同索引策略，而非将所有内容放进同一个 FTS 表。

## 6 典型应用

- **个人记忆与本地知识库**：索引笔记、会话摘要、网页剪藏和元数据，按关键词或短语快速回忆。
- **应用内搜索**：为桌面、移动或离线应用提供轻量全文搜索。
- **技术文档与日志检索**：检索 API 名称、错误码、文件路径和配置项。
- **RAG 的稀疏召回**：将 FTS5 的 BM25 Top-K 与向量检索结果去重，再用 RRF 融合和 reranker 精排。

RAG 场景中不要直接将 FTS5 的 BM25 分数与向量相似度相加：两者的量纲和方向不同。缺少标注数据进行分数校准时，优先采用只依赖排名的 RRF。
