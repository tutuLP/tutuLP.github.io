图片处理：

```
前端生成 upload_session_id
        ↓
用户粘贴 / 选择图片
        ↓
前端上传图片（带 upload_session_id）
        ↓
后端存储图片（status = temp）
        ↓
后端返回 { asset_id, url }
        ↓
前端插入 Markdown: ![](url)

==== 编辑中（随便删、撤销、重来） ====

用户点击「保存文章」
        ↓
前端提交 { article_id?, markdown, upload_session_id }
        ↓
后端解析 markdown 中图片
        ↓
【转正】markdown 中引用的 temp 图片
        ↓
【回收】该 session 中未被引用的 temp 图片
        ↓
文章保存完成

```

```
CREATE TABLE image_asset (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  url VARCHAR(1024) NOT NULL,
  storage_key VARCHAR(512) NOT NULL,
  upload_session_id CHAR(36) NOT NULL,
  status ENUM('temp','active') NOT NULL,
  ref_count BIGINT NOT NULL DEFAULT 0,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  activated_at DATETIME DEFAULT NULL,
  deleted_at DATETIME DEFAULT NULL,
  INDEX idx_session_status (upload_session_id, status),
  UNIQUE KEY uniq_url (url)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

```

axum（HTTP 网关/BFF） + tonic（gRPC/RPC） + tower（超时/重试/限流/熔断/追踪） + tracing/opentelemetry（观测）+sqlx+query_as

[dependencies]
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
sqlx = { version = "0.7", features = ["runtime-tokio", "mysql", "macros"] }

```
use sqlx::{MySql, MySqlPool, QueryBuilder};

#[derive(Debug, Clone, sqlx::FromRow)]
struct News {
    id: i64,
    title: String,
    content: String,
    review: bool,
}

async fn insert_news(pool: &MySqlPool, items: &[News]) -> sqlx::Result<()> {
    if items.is_empty() {
        return Ok(());
    }

    let mut qb: QueryBuilder<MySql> =
        QueryBuilder::new("INSERT INTO news_telegraph (id, title, content, review) ");

    qb.push_values(items, |mut b, item| {
        b.push_bind(item.id)
            .push_bind(&item.title)
            .push_bind(&item.content)
            .push_bind(item.review);
    });

    // 如果你担心重复主键，可以按需加：ON DUPLICATE KEY UPDATE ...
    // qb.push(" ON DUPLICATE KEY UPDATE ")
    //   .push("title=VALUES(title), content=VALUES(content), review=VALUES(review)");

    qb.build().execute(pool).await?;
    Ok(())
}

async fn query_news_by_review(pool: &MySqlPool, review: bool) -> sqlx::Result<Vec<News>> {
    let rows = sqlx::query_as::<_, News>(
        r#"
        SELECT id, title, content, review
        FROM news_telegraph
        WHERE review = ?
        ORDER BY id
        "#,
    )
    .bind(review)
    .fetch_all(pool)
    .await?;

    Ok(rows)
}

#[tokio::main]
async fn main() -> sqlx::Result<()> {
    // 你可以用环境变量：DATABASE_URL="mysql://user:pass@127.0.0.1:3306/dbname"
    let database_url = std::env::var("DATABASE_URL")
        .expect("missing DATABASE_URL, e.g. mysql://root:pass@127.0.0.1:3306/test");

    let pool = MySqlPool::connect(&database_url).await?;

    // 准备测试数据（同一个结构体用于 insert 和 query）
    let items = vec![
        News {
            id: 1,
            title: "如何提升睡眠质量".to_string(),
            content: "从规律作息、减少咖啡因开始...".to_string(),
            review: false,
        },
        News {
            id: 2,
            title: "苹果手机拍照技巧".to_string(),
            content: "夜景模式、构图、RAW...".to_string(),
            review: true,
        },
    ];

    // 写入：QueryBuilder<MySql>
    insert_news(&pool, &items).await?;

    // 查询：query_as::<_, News>()
    let reviewed = query_news_by_review(&pool, true).await?;
    println!("review=true rows: {reviewed:#?}");

    Ok(())
}

```

`query_as::<_, News>()` **必须 SELECT 出 News 结构体所需的全部字段**（除非字段用 `Option<T>`）

MySQL 的 `BOOLEAN` 一般映射到 Rust `bool` 没问题。



# 搜索

```
1. Query 预处理
   - 分词，非侵入式 Query 清洗(去多余空格,统一大小写（英文）,全角半角统一,去特殊符号)

2. 并行召回（Recall）
   2.1 BM25 关键词召回（OpenSearch）
       → TopK₁（例如 500～2000）
   2.2 Dense 向量召回（Milvus）
       → TopK₂（例如 200～1000）

3. 候选集合融合（Merge / Fuse）
   - 去重
   - 初步归一化（score normalize）
   - 规则融合（RRF）

4. 重排序（Re-ranking)
   - Cross-Encoder
   - TopN（例如 20～100）

5. 后处理（Post-processing）
   - 关键词与语义高亮

6. 返回结果

```



 **“索引层 + 召回层 + 融合排序层 + 业务过滤层 + 缓存与稳定性层”** 五件事来做。
“标题/正文/标签联合搜索”，本质是 **多字段、多路召回 + 统一融合排序**。

下面给你一套在 **超大数据量、超高并发** 下能落地的方案。

------

## 1) 数据与索引的总体架构

### 数据层

- PostgreSQL：保留元数据
- **对象存储（原文）**：md格式+图片

### 搜索索引层（两类索引，缺一不可）

1. **全文索引（BM25）**：负责“关键词精准召回”
   - ES / OpenSearch（最常见）
2. **向量索引（Dense）**：负责“语义召回”
   - Milvus 2.5（企业级主流）

> 企业级“联合搜索”的核心：**BM25 + Dense 并行召回**，再融合。再重排序

------

## 2) 索引设计：标题/摘要/正文怎么建？

### 文档粒度建议：Article + Chunk 双层

- **Article 索引（文章级）**
  - 向量：`title_vec`
  - BM25：title、tag？
- **Chunk 索引（段落级）
  - 向量：`chunk_vec`（主力）
  - BM25：chunk_text（正文召回用）

### 多字段 BM25 的“权重”怎么做？

典型权重（经验值，可调）：

- title：3.0
- summary：2.0
- content chunk：1.0

ES/OpenSearch 用 `multi_match` + boosts；

------

## 3) 查询流程：企业级联合搜索怎么跑？

把一次查询拆成 4 步（高并发下非常稳定）：

### Step A：候选过滤（必须先做）

按业务强过滤减少召回压力：

- 时间范围（最近 7/30/365 天）
- 语言/地区（可选）
- 状态
- tag

> 过滤要尽量在索引层完成（减少候选规模），别把海量候选拉回服务层再过滤。

------

### Step B：并行多路召回（Recall）

对同一个 query，并行跑 2～4 路召回：

1. **BM25(title + summary + chunk_text)**：取 TopN（例如 500）
2. **Dense(title_vec)**：取 TopN（例如 200）
3. **Dense(chunk_vec)**：取 TopN（例如 500）
4. （可选）**Dense(summary_vec)**：取 TopN（例如 200）

> 超高并发下，TopN 不要太小（避免漏召回），也不要太大（避免融合成本爆炸）。新闻场景常用 200～1000。

------

### Step C：融合排序（Fusion）

企业级最稳的融合方法是 **RRF（Reciprocal Rank Fusion）**：

也可以做加权融合（需要做分数归一化）：

- `score = 0.45*bm25 + 0.15*title_dense + 0.40*chunk_dense`

经验：

- 短 query（关键词）提高 BM25 权重
- 长 query（描述）提高 chunk_dense 权重

------

### Step D：精排（Rerank）与聚合（重要）

#### 1) Chunk → Article 聚合

chunk 召回的是段落，最终要返回“新闻”：

- 按 article_id 分组
- 取每篇文章最强的 1～3 个 chunk 作为 evidence/snippet
- article 的最终分 = max(chunk_score) 或 top2 平均

#### 2) 轻量精排（高并发友好）

对融合后的 TopK（比如 100）做一次 rerank：

- Cross-encoder reranker（效果最好，但成本高）
- 或轻量 rerank：基于实体命中、时间新鲜度、来源权重、是否标题命中等特征的 Learning-to-Rank

企业级通常是：

- 热门 query：rerank（可以）
- 超高 QPS：只对 Top20 rerank，或按条件触发（长 query 才 rerank）

------

## 4) 分片与存储：超大数据量怎么扛？

新闻是典型的“时间序列型内容”，最佳实践：

- **按时间分区/分片**：monthly/quarterly shards
- **热冷分层**：
  - 热（近 90 天）索引全量驻留、QPS 扛大头
  - 温（1～2 年）索引可查但资源降配
  - 冷（更老）用更便宜存储/按需加载或降低召回路数

Milvus/ES 都能做类似分片与生命周期管理（ILM）。

------

## 5) 写入与更新：高吞吐索引怎么维护？

企业级几乎都用 **CDC / 事件流**：

- MySQL → Debezium/Canal → Kafka
- Indexer 服务消费 Kafka：
  - 文本清洗、chunk
  - embedding（GPU/批处理）
  - 写 BM25 索引 + 写向量索引
- 删除/更新用同一个流水线（保证一致性）
- 失败重试、幂等（按 id / version）

关键点：

- embedding 必须批处理（吞吐高很多）
- 索引写入要支持幂等 upsert（version/updated_at）

------

## 6) 缓存与稳定性：超高并发必备

- **Query cache**：热门 query 结果缓存（Redis/本地 cache），TTL 30s～5min
- **Embedding cache**：相同 query 不重复算向量
- **降级策略**：
  - 向量服务压力大：只走 BM25
  - BM25 压力大：只走 dense
  - rerank 压力大：跳过 rerank

------

## 7) 你要的“标题/正文/摘要联合搜索”最终形态

最终对外一个接口，但内部多路召回：

- BM25：title+summary+chunk_text（权重不同）
- Dense：title_vec + chunk_vec（权重不同）
- 融合：RRF 或加权
- 聚合：chunk → article
- 返回：标题 + 时间 + 来源 + 命中摘要/命中段落 snippet + score

------

## 推荐落地组合

如果你要“企业级、超大规模、超高并发”并且想一次做对：

- **ES/OpenSearch（BM25、多字段权重、过滤、聚合强）**
- **Milvus 2.5（Dense 向量、可加 Sparse 做 hybrid）**
- **Chunk 化正文**（必做）
- **RRF 融合 + TopN rerank**（平衡质量与成本）
- **CDC(Kafka) 索引流水线**（稳定可扩展）

------

如果你告诉我两件事，我可以把“字段设计 + 分片策略 + 召回参数 + 融合公式 + 接口返回结构”直接给成一套可实现规格：

1. 数据规模：每天新增多少条？历史总量多少？
2. QPS 目标：峰值查询 QPS 大概多少（100/1k/10k+）？



### 路线 A：OpenSearch 做 BM25 + Milvus 做 Dense（最稳、功能最全）

**OpenSearch 里存一条“文章级文档”**（用于字段检索/过滤/聚合）：

- article_id
- title（BM25）
- author_name（BM25/keyword）
- tags（keyword + text）
- toc_text（text，目录拼起来）
- body_text（可选：正文全量 text；或只存摘要）
- publish_time / updated_at
- 其他过滤字段（状态、空间、权限等）

同时你再存一条“chunk 级文档”（可选但推荐，便于正文命中片段）：

- chunk_id, article_id
- chunk_text
- heading_path（该 chunk 属于哪个目录）
- tags/author（冗余一份用于过滤）

**Milvus 里存 chunk 的 dense 向量**（语义召回 + 片段返回）：

- chunk_id, article_id
- dense_vector(512)
- 过滤字段：author_id、tag_ids、time、status…

查询时：

- OpenSearch：字段 BM25（title/tags/author/toc/body）+ filter
- Milvus：dense 召回 + filter
- 应用层融合：按你之前说的比例聚合排序（title/tags/toc/body/chunk）并返回片段

这条路线最像“知乎/CSDN/知识库产品”的工程形态。

# 对象存储

MinIO

草稿箱
状态字段

上传检测是否违规，检查文字和图片，标记状态，用户可在草稿箱中看到未通过的文章，也能看到未发布的文章

上传-违规检测-三级目录分析-tag生成-内容格式自动排版-修改建议

使用ai的时候需要记录文章id，下一次恢复多轮对话



# 架构

```
前期：10万用户以内
Nginx+Go(Gin)

后期：
       [ Kong API Gateway ]
  (认证-JWT/OIDC / 限流 / 反爬 / 路由 / 审计)
                 ↓
        [ Go - Gin API 层 ]
                 ↓
                gRPC
```

环境：
Centos Podman Nginx PostgreSQL Go

> 其中PostgreSQL Nginx不放在容器Podman中，而后续的Go服务等需要放在容器中

*为什么go程序建议运行在podman中？*

环境隔离与依赖，升级，回滚，迁徙，CI/CD等方面更加方便



# 向量选择

https://huggingface.co/spaces/mteb/leaderboard

排行第一的：Kingsoft-LLM/QZhou-Embedding-Zh



`HF_ENDPOINT=https://hf-mirror.com python export_model.py`









OpenSearch

Dense+bm25 向量索引+倒排索引

RRF

rerank

```
{
  "源站": "web_finance_news_db_20260107",
  "原正文链接": "https://m.jiemian.com/article/8678380.html",
  "源站上的新闻时间": "2023-01-03 13:26:00",
  "抓取时间": "2026-01-07 08:25:39",
  "标题": "",
  "正文": ""
}
```

bm25：标题和正文都做分词



sudo sysctl -w vm.max_map_count=262144

echo "vm.max_map_count=262144" >> /etc/sysctl.conf

mkdir -p /data/opensearch/node1
mkdir -p /data/opensearch/node2

chmod -R 777 /data/opensearch

```

  podman run -d \
  --name opensearch-node1 \
  -p 9200:9200 \
  -v /data/opensearch/node1:/usr/share/opensearch/data:Z \
  -e "discovery.type=single-node" \
  -e "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m" \
  -e "DISABLE_SECURITY_PLUGIN=true" \
  docker.io/opensearchproject/opensearch:3.5.0
  
curl http://localhost:9200/_cluster/health?pretty
    
  podman run -d \
  --name opensearch-dashboards \
  -p 5601:5601 \
  -e "OPENSEARCH_HOSTS=[\"http://host.containers.internal:9200\"]" \
  -e "DISABLE_SECURITY_DASHBOARDS_PLUGIN=true" \
  docker.io/opensearchproject/opensearch-dashboards:3.5.0
  
  http://localhost:5601
  
  
  podman run -d \
  --name opensearch-node1 \
  -p 9200:9200 \
  -e "discovery.type=single-node" \
  -e "OPENSEARCH_JAVA_OPTS=-Xms1g -Xmx1g" \
  -e "DISABLE_SECURITY_PLUGIN=true" \
  -e "action.auto_create_index=false" \
  -e "cluster.routing.allocation.disk.threshold_enabled=false" \
  docker.io/opensearchproject/opensearch:latest
```

<img src="C:\MY\md\tutuLP.github.io\杂说\images\markdown.assets\image-20260320210845455.png" alt="image-20260320210845455" style="zoom:33%;" />

存储策略：

> opensearch应该需要检索的全文数据而不是像milvs只存向量

OpenSearch = 搜索引擎（全文 + 向量）
 PostgreSQL = 数据源

psql -h 127.0.0.1 -p 5432 -U postgres
mysecretpassword


CREATE TABLE news_meta (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    source          TEXT,                       -- 源站
    url             TEXT UNIQUE,                         -- 原文链接（去重关键）
    publish_time    TIMESTAMPTZ,                         -- 新闻发布时间（建议带时区）
    crawl_time      TIMESTAMPTZ DEFAULT NOW(),           -- 抓取时间
    title           TEXT NOT NULL,                       -- 标题
    file_path       TEXT                                 -- 文件路径（如按月分桶）
);
CREATE INDEX idx_news_publish_time
ON news_meta (publish_time DESC);

JSON → 批量解析 → CSV → PostgreSQL COPY
COPY → 临时表 → INSERT ON CONFLICT DO NOTHING
pip install psycopg2-binary tqdm

-- 关闭同步日志（提升速度）
SET synchronous_commit = OFF;

-- 临时关闭自动分析
ALTER TABLE news_meta SET (autovacuum_enabled = false);
-- 恢复
ALTER TABLE news_meta SET (autovacuum_enabled = true);

VACUUM ANALYZE news_meta;

DROP TABLE IF EXISTS news_meta_tmp;
CREATE TABLE news_meta_tmp (
    source          TEXT,
    url             TEXT,
    publish_time    TIMESTAMPTZ,
    crawl_time      TIMESTAMPTZ,
    title           TEXT,
    file_path       TEXT
);


```

```python
import os
import json
import psycopg2
from multiprocessing import Pool, cpu_count
from tqdm import tqdm
import tempfile

DATA_DIR = "/root/news_data/2023"
BATCH_SIZE = 5000

DB_CONFIG = {
    "host": "127.0.0.1",
    "port": 5432,
    "user": "postgres",
    "password": "mysecretpassword",
    "dbname": "postgres"
}


def safe_time(val):
    """简单清洗时间，避免非法值导致 COPY 失败"""
    if not val:
        return None
    try:
        return val.replace("T", " ").strip()
    except:
        return None


def parse_file(file_path):
    try:
        with open(file_path, "r", encoding="utf-8") as f:
            data = json.load(f)

        title = data.get("标题", "").strip()
        if not title:
            return None  # 过滤标题为空

        return (
            data.get("源站"),
            data.get("原正文链接"),
            safe_time(data.get("源站上的新闻时间")),
            safe_time(data.get("抓取时间")),
            title,
            file_path
        )
    except Exception:
        return None


def collect_files():
    all_files = []
    for root, _, files in os.walk(DATA_DIR):
        for f in files:
            if f.endswith(".json"):
                all_files.append(os.path.join(root, f))
    return all_files


def write_batch_to_csv(rows):
    tmp = tempfile.NamedTemporaryFile(delete=False, mode="w", encoding="utf-8")

    for r in rows:
        line = "\t".join([
            str(x).replace("\n", " ").replace("\t", " ") if x else "\\N"
            for x in r
        ])
        tmp.write(line + "\n")

    tmp.close()
    return tmp.name


def copy_to_pg(csv_file):
    conn = psycopg2.connect(**DB_CONFIG)
    cur = conn.cursor()

    with open(csv_file, "r", encoding="utf-8") as f:
        cur.copy_expert("""
            COPY news_meta_tmp (
                source,
                url,
                publish_time,
                crawl_time,
                title,
                file_path
            )
            FROM STDIN WITH (
                FORMAT text,
                DELIMITER E'\\t',
                NULL '\\N'
            )
        """, f)

    # 去重插入
    cur.execute("""
        INSERT INTO news_meta (
            source, url, publish_time, crawl_time, title, file_path
        )
        SELECT source, url, publish_time, crawl_time, title, file_path
        FROM news_meta_tmp
        ON CONFLICT (url) DO NOTHING;
        
        TRUNCATE news_meta_tmp;
    """)

    conn.commit()
    cur.close()
    conn.close()

    os.remove(csv_file)


def main():
    files = collect_files()
    print(f"Total files: {len(files)}")

    pool = Pool(cpu_count())

    batch = []
    for result in tqdm(pool.imap_unordered(parse_file, files), total=len(files)):
        if result:
            batch.append(result)

        if len(batch) >= BATCH_SIZE:
            csv_file = write_batch_to_csv(batch)
            copy_to_pg(csv_file)
            batch.clear()

    # 最后一批
    if batch:
        csv_file = write_batch_to_csv(batch)
        copy_to_pg(csv_file)

    pool.close()
    pool.join()


if __name__ == "__main__":
    main()
```



文档索引（BM25）news_index

```json
{
  "id": "123",
  "title": "...",
  "content": "...",
  "publish_time": "...",
  "source": "..."
}
```

 chunk 索引（向量）news_chunk_index

```json
{
  "doc_id": "123",
  "chunk_id": "123_1",
  "content_chunk": "...",
  "embedding": [...]
}
```



BM25 查询（带高亮）

```
curl -X POST "http://localhost:9200/news_index/_search" \
-H "Content-Type: application/json" \
-d '{
  "size": 50,
  "_source": ["id", "title"],
  "query": {
    "multi_match": {
      "query": "美联储加息",
      "fields": ["title^3", "content"],
      "type": "best_fields"
    }
  },
  "highlight": {
    "pre_tags": ["<em>"],
    "post_tags": ["</em>"],
    "fields": {
      "title": {},
      "content": {
        "fragment_size": 150,
        "number_of_fragments": 1
      }
    }
  }
}'
```

向量搜索（chunk）

```
curl -X POST "http://localhost:9200/news_chunk_index/_search" \
-H "Content-Type: application/json" \
-d '{
  "size": 50,
  "_source": ["doc_id", "chunk_id", "content_chunk"],
  "query": {
    "knn": {
      "embedding": {
        "vector": [/* 你的向量 */],
        "k": 50,
        "num_candidates": 100
      }
    }
  }
}'
```

RRF融合

`score = Σ (1 / (60 + rank))`

```
let mut doc_map = HashMap<doc_id, Result>();

// BM25
for (rank, doc) in bm25_results {
    let score = 1.0 / (60.0 + rank);

    doc_map[doc.id].score += score;
    doc_map[doc.id].title = doc.title;
    doc_map[doc.id].bm25_highlight = doc.highlight;
}

// 向量 chunk
for (rank, chunk) in vector_results {
    let score = 1.0 / (60.0 + rank);

    doc_map[chunk.doc_id].score += score;

    doc_map[chunk.doc_id].chunks.push(chunk);
}
```

chunk->doc

`best_chunk = max_score_chunk(doc.chunks)`

最终返回

```
{
  "doc_id": "123",
  "title": "美联储加息影响市场",
  "score": 0.92,

  "snippet": "美联储加息导致市场波动...",   // 来自 chunk

  "highlight": [
    "...<em>美联储</em><em>加息</em>..."
  ]
}
```

可以在chunk中存：
{
  "chunk_id": "123_2",
  "start_offset": 500,
  "end_offset": 900
}
方便定位原文位置，或者直接按照chunk_id来算位置

- bge-reranker-base
- bge-reranker-large
- m3e-reranker

为啥没有单独存标题向量？

> 标题已经融合在每一个chunk中，多一轮搜索性能下降

什么时候“必须要 title 向量”？

> 短 query,正文“废话多”,发现问题：明明有相关文章，但没召回



IK 分词插件

```
podman exec -it opensearch bash

./bin/opensearch-plugin install \
https://get.infini.cloud/opensearch/analysis-ik/3.6.0

podman restart opensearch

curl -X GET "localhost:9200/_analyze" -H 'Content-Type: application/json' -d'
{
  "analyzer": "ik_smart",
  "text": "美联储加息对市场的影响"
}'
```



```shell
pg中存储：
psql -h 127.0.0.1 -p 5432 -U postgres
mysecretpassword
CREATE TABLE news_meta (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    source          TEXT,                       -- 源站
    url             TEXT UNIQUE,                         -- 原文链接（去重关键）
    publish_time    TIMESTAMPTZ,                         -- 新闻发布时间（建议带时区）
    crawl_time      TIMESTAMPTZ DEFAULT NOW(),           -- 抓取时间
    title           TEXT NOT NULL,                       -- 标题
    file_path       TEXT                                 -- 文件路径（如按月分桶）
);
源文件：
A1016194417.json
{
  "源站": "web_finance_news_db_20260107",
  "原正文链接": "https://m.jiemian.com/article/8678380.html",
  "源站上的新闻时间": "2023-01-03 13:26:00",
  "抓取时间": "2026-01-07 08:25:39",
  "标题": "",
  "正文": ""
}

1. 读取数据
2. chunk切分
   chunk1: 0-300
   chunk2: 250-550
   chunk3: 500-800
   ...

3. 生成 embedding 标题+chunk
4. 写入数据
要求：批量+并发
✅ PostgreSQL 读取

✅ JSON 文件解析（你这种字段名）

✅ chunk 滑窗切分（300 / overlap=50）

✅ embedding（标题 + chunk 拼接）

✅ OpenSearch bulk 写入

✅ 批量 + 并发（线程池）

✅ 高性能（批处理 + 减少 IO）

curl -X POST "http://115.159.79.101:9200/news_index/_bulk" \
-H "Content-Type: application/json" \
-d '
{ "index": { "_id": "doc_1" } }
{ "id": "doc_1", "title": "美联储加息影响全球市场", "content": "完整新闻内容...", "publish_time": "2024-01-01T10:00:00Z", "source": "Reuters", "create_time": "2026-03-21T12:00:00Z" }
{ "index": { "_id": "doc_2" } }
{ "id": "doc_2", "title": "AI行业迎来新突破", "content": "OpenAI发布新模型...", "publish_time": "2024-01-02T10:00:00Z", "source": "TechNews", "create_time": "2026-03-21T12:00:00Z" }
'

curl -X POST "http://localhost:9200/news_chunk_index/_bulk" \
-H "Content-Type: application/json" \
-d '
{ "index": { "_id": "doc_1_0" } }
{ "doc_id": "doc_1", "chunk_id": "doc_1_0", "content_chunk": "美联储宣布加息", "embedding": [0.12, 0.98, ...] }
{ "index": { "_id": "doc_1_1" } }
{ "doc_id": "doc_1", "chunk_id": "doc_1_1", "content_chunk": "市场出现波动", "embedding": [0.33, 0.21, ...] }
'

pip install -i https://pypi.tuna.tsinghua.edu.cn/simple \
psycopg2-binary \
requests \
numpy \
onnxruntime \
transformers \
tqdm

nohup python insert4.py > run.log 2>&1 &

curl -X GET "http://localhost:9200/_cat/indices?v"
curl -X GET "http://localhost:9200/news_index/_count"
curl -X GET "http://localhost:9200/news_chunk_index/_count"

ps aux | grep insert.py

1. 下载cuda12
https://developer.nvidia.com/cuda-12-0-0-download-archive?target_os=Windows&target_arch=x86_64&target_version=Server2022&target_type=exe_local
2. 选择的安装位置
C:\Users\tutu\AppData\Local\cuda
3. 下载 cudnn
下载链接：https://developer.download.nvidia.cn/compute/cudnn/redist/cudnn/windows-x86_64/
下载版本：cudnn-windows-x86_64-9.20.0.48_cuda12-archive.zip
%CUDA_PATH% 环境变量指向 CUDA 安装目录
%PATH% 中包含 .../CUDA/v12.x/bin 和 .../CUDA/v12.x/lib64（Windows 需添加到 PATH）
不要选择下述的exe版本
https://developer.nvidia.com/cudnn-downloads?target_os=Windows&target_arch=x86_64&target_version=Server2022&target_type=exe_local
4. powershell中输入 $env:CUDA_PATH 查看cuda位置：C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v12.0
将cudnn解压的内容放入cuda文件夹
(1) bin\cudnn64_9.dll → CUDA\v12.0\bin\
(2) include\* → CUDA\v12.0\include\
(3) lib\x64\* → CUDA\v12.0\lib\x64\
5. PATH中新增下述环境变量
C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v12.0\bin
C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v12.0\lib\x64
6. 重启电脑 where cudnn64_9.dll
python -c "import ctypes; ctypes.CDLL('cudnn64_9.dll')" 不报错就行了
python -c "import ctypes; ctypes.CDLL('C:\\Program Files\\NVIDIA\\CUDNN\\v9.20\\bin\\12.9\\x64\\cudnn64_9.dll')"

https://www.winimage.com/zLibDll/zlib123dllx64.zip  https://zlib.net/
dll_x64/zlibwapi.dll
C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v12.0\bin

12.9

pip uninstall onnxruntime -y
pip install onnxruntime-gpu

```

curl -X DELETE "http://localhost:9200/news_chunk_index"

```shell
curl -X PUT "http://localhost:9200/news_index" -H "Content-Type: application/json" -d '{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "refresh_interval": "30s",
    "analysis": {
      "analyzer": {
        "ik_max_word_analyzer": {
          "type": "custom",
          "tokenizer": "ik_max_word"
        },
        "ik_smart_analyzer": {
          "type": "custom",
          "tokenizer": "ik_smart"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "id": { "type": "keyword" },
      "title": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "content": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart"
      },
      "publish_time": { "type": "date" },
      "source": { "type": "keyword" },
      "create_time": { "type": "date" }
    }
  }
}'

curl -X PUT "http://localhost:9200/news_chunk_index" \
-H "Content-Type: application/json" \
-d '{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1,
    "refresh_interval": "60s",
    "index": {
      "knn": true
    }
  },
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "doc_id": { 
        "type": "keyword" 
      },
      "chunk_id": { 
        "type": "keyword" 
      },
      "content_chunk": {
        "type": "text",
        "analyzer": "ik_max_word"
      },
      "embedding": {
        "type": "knn_vector",
        "dimension": 1024,
        "method": {
          "name": "hnsw",
          "engine": "lucene",
          "space_type": "cosinesimil",
          "parameters": {
            "ef_construction": 256,
            "m": 48
          }
        }
      }
    }
  }
}'

curl -X GET "http://localhost:9200/news_chunk_index/_mapping?pretty"

podman run -d \
  --name opensearch-node1 \
  -p 9200:9200 \
  -v /data/opensearch/node1:/usr/share/opensearch/data:Z \
  -e "discovery.type=single-node" \
  -e "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m" \
  -e "DISABLE_SECURITY_PLUGIN=true" \
  -e "NETWORK_HOST=0.0.0.0" \
  docker.io/opensearchproject/opensearch:3.5.0
```

![image-20260404000406008](C:\MY\md\tutuLP.github.io\杂说\images\markdown.assets\image-20260404000406008.png)

![image-20260404001752179](C:\MY\md\tutuLP.github.io\杂说\images\markdown.assets\image-20260404001752179.png)



删除索引并重建

curl -X GET "http://localhost:9200/news_index/_count"
curl -X GET "http://localhost:9200/news_chunk_index/_count"
