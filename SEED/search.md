# 搜索

### 搜索引擎

直接在数据库中搜索+全文搜索引擎

- 全文搜索引擎

  中型方案：Meilisearch
  大型方案：Elasticsearch，分词使用ik_max_word

```json
{
  "mappings": {
    "properties": {
      "title":    { "type": "text", "analyzer": "ik_max_word" },
      "content":  { "type": "text", "analyzer": "ik_max_word" },
      "tags":     { "type": "keyword" },
      "category": { "type": "keyword" }
    }
  }
}
```

1. 文档采集，提取文件名
2. 文本提取，textract，`chardet` + `open()`，输出纯文本
3. 分词与预处理，文本切分为可索引的关键词，中文：jieba，thulac，英文：分词器（Tokenizer）、去除停用词（Stop Words）、词干提取（Stemming）
4. 索引构建，将所有词映射到文件及位置{
   "搜索": {"doc1.txt": [12, 57], "[doc2.md](http://doc2.md/)": [33]},
   "引擎": {"doc1.txt": [13]}
   }
5. 查询处理与匹配：用户输入关键词 → 同样进行分词 → 匹配倒排索引 → 返回结果

全 md，千万级文本搜索方案：

| 模块     | 推荐组件                         | 理由                                      |
| -------- | -------------------------------- | ----------------------------------------- |
| 文本解析 | 自定义 parser / `markdown-it`    | 只提取正文，忽略链接/图片等               |
| 分词     | `jieba` (Python) / `tokenizers`  | 支持中文/英文混合，简洁高效               |
| 索引引擎 | ✅ [Tantivy (Rust)] / Meilisearch | 适合本地嵌入，性能优越，支持百万~亿级文档 |
| 数据结构 | 倒排索引（inverted index）       | 基础搜索引擎结构                          |
| 查询系统 | REST API / CLI 工具              | 适配桌面应用或脚本                        |

**Tantivy + Rust**（超高性能本地搜索）

Tantivy 是一个用 Rust 编写的搜索引擎库，相当于本地 Lucene。

数据建模

```jsx
每个 md 索引字段：

{
"path": String,               // 文件路径
"title": String,              // 文件名或第一个 # 标题
"content": String,            // 正文纯文本（过滤掉 markdown 标记）
"updated_at": i64             // 文件修改时间
}
```

文本提取：使用 [`pulldown-cmark`](https://github.com/raphlinus/pulldown-cmark) 解析 Markdown

```jsx
use pulldown_cmark::{Parser, Event};

fn extract_text_from_md(markdown_input: &str) -> String {
    let parser = Parser::new(markdown_input);
    parser
        .filter_map(|event| match event {
            Event::Text(text) => Some(text.to_string()),
            _ => None,
        })
        .collect::<Vec<_>>()
        .join(" ")
}

```

Tantivy 索引创建

```jsx
let mut schema_builder = Schema::builder();
schema_builder.add_text_field("title", TEXT | STORED);
schema_builder.add_text_field("content", TEXT);
schema_builder.add_text_field("path", STORED);
schema_builder.add_date_field("updated_at", STORED);
let schema = schema_builder.build();

// 构建索引 & 写入文档
let index = Index::create_in_dir("tantivy_index", schema.clone())?;
let mut index_writer = index.writer(50_000_000)?; // 50MB 缓冲

index_writer.add_document(doc!(
    title => "文档标题",
    content => "纯文本内容",
    path => "/path/to/file.md",
    updated_at => DateTime::from_utc(…),
));

```

查询逻辑：

```jsx
let reader = index.reader()?;
let searcher = reader.searcher();
let query_parser = QueryParser::for_index(&index, vec![title, content]);

let query = query_parser.parse_query("人工智能 石油")?;
let top_docs = searcher.search(&query, &TopDocs::with_limit(10))?;

for (_score, doc_address) in top_docs {
    let retrieved_doc = searcher.doc(doc_address)?;
    println!("{}", schema.to_json(&retrieved_doc));
}

```

增量索引与文件监控

```jsx
使用 notify（Rust）监听文件夹变更，进行增量索引。

文件修改时间戳可用于判断是否需要重新索引
```

索引存储建议

```jsx
每 10 万 ~ 50 万文档一个独立索引分片目录。
可采用多线程并行创建索引（Rust 的 rayon）。
合并搜索结果时使用分布式 Top-N 聚合。
```

性能预估：

```jsx
| 数据规模       | 建议结构       | 预计耗时（索引）  | 查询速度（Top 10） |
| ---------- | ---------- | --------- | ------------ |
| 100 万文档    | 1 个索引      | 1\~3 分钟   | <50 ms       |
| 1000 万文档   | 10 个分片     | 10\~20 分钟 | 100\~300 ms  |
| 5000 万文档以上 | 分片 + 分布式合并 | 支持实时或批量更新 | <1 秒         |

```

替代方案：

```jsx
| 框架                  | 优点                  | 缺点              |
| ------------------- | ------------------- | --------------- |
| **Tantivy (Rust)**  | 高性能、本地部署、嵌入式支持      | 编写需用 Rust       |
| **Meilisearch**     | REST API、易部署、中文支持较好 | 内存占用高，不适合超大本地部署 |
| **Lucene / Solr**   | 成熟、功能全              | 资源开销大，需 JVM     |
| **Whoosh (Python)** | 简单易上手               | 仅适合小型文档集        |

```

总结：**Rust + Tantivy + pulldown-cmark + 自定义分词器** 

建议：

1. 分词器：使用 Transformer 模型（如 BERT、RoBERTa）来理解句法结构，进行上下文相关的分词与词性分析。**适合中文语义分词**：使用如百度 ERNIE、哈工大 LTP、HanLP 等
2. Embedding 语义向量索引

并行构建一个**向量索引（semantic index）**，例如使用 `Qdrant` / `Milvus` / `Weaviate` 或 `tantivy + faiss` 实现向量搜索。

将 Markdown 文档内容通过 LLM 编码为向量，用户查询也向量化后进行**语义相似度匹配**

1. 结果摘要：

对于搜索命中的文档，使用本地 LLM 模型（如 Mistral、Phi-2）或者 distilBERT 来生成文档的语义摘要

注意：优先级     标题>tag>正文

对输入的问题进行分词，找到对应的文章标题，问题分词再匹配 tag

问题:[windows] [安装] [redis] [1.0] 

问题：[vmware] [15] [安装] [centos] [7]

找到文章：windwos 安装 redis，匹配标签  redis 1.0

找到文章：vmware 安装 centos，匹配标签 vmware15，centos7

根据主题也可以对应教程，比如wmware 中安装 centos这个教程既会存在 wmware 板块也会存在 centos 板块。