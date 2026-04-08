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

```
A1016194417.json
{
  "源站": "web_finance_news_db_20260107",
  "原正文链接": "https://m.jiemian.com/article/8678380.html",
  "源站上的新闻时间": "2023-01-03 13:26:00",
  "抓取时间": "2026-01-07 08:25:39",
  "标题": "格林美：已在2021年完成韩国浦项动力电池回收基地2万吨废旧电池处理产线建设和运营",
  "正文": "格林美：已在2021年完成韩国浦项动力电池回收基地2万吨废旧电池处理产线建设和运营 | 界面新闻\n注册/登录\n个人中心\n首页\n天下\n中国\n宏观\n文娱\n体育\n时尚\n文化\n旅行\n生活\n游戏\n歪楼\n数据\n正午\n艺术\n商业\n科技\n汽车\n地产\n证券\n金融\n消费\n工业\n交通\n投资\n营销\n职场\n管理\n创业\n出国\n楼市\n财富\n健康\n大湾区\n酒业\n区块链\nESG\n智库\n更多\n快讯\n热文\n视频\n图片\n召集令\n专题\n直播\nA     APP下载\n格林美：已在2021年完成韩国浦项动力电池回收基地2万吨废旧电池处理产线建设和运营\n格林美1月3日在互动平台表示，公司已在2021年完成韩国浦项动力电池回收基地2万吨废旧电池处理产线的建设和运营。目前韩国回    回收工厂运营良好。\n广告等商务合作，\n请点击这里\n未经正式授权严禁转载本文，侵权必究。\n打开界面新闻APP，查看原文\n打开界面新闻，查看更多专业报道\n相关推荐\n热门评论\n打开APP，查看全部评论，抢神评席位\n热    热门推荐\n更多精彩内容\n下载界面APP 订阅更多品牌栏目\n关注界面\n进入首页\nAPP下载\n界面新闻\n只服务于独立思考的人群\n打开"
}
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



```
用户 query
   ↓
① BM25（doc）
② 向量（chunk）
③ （可选）title向量
   ↓
④ RRF 融合（Top 50）
   ↓
⑤ Cross-Encoder rerank（Top 20 → Top 5）
   ↓
⑥ 返回结果

后续优化：Query 改写，多向量，用户点击训练排序模型，RAG
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

```
PUT news_index
{
  "settings": {
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
        "search_analyzer": "ik_smart"
      },

      "content": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart"
      },

      "publish_time": { "type": "date" },
      "source": { "type": "keyword" }
    }
  }
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

```
PUT news_chunk_index
{
  "settings": {
    "index": {
      "knn": true
    }
  },
  "mappings": {
    "properties": {
      "doc_id": { "type": "keyword" },
      "chunk_id": { "type": "keyword" },

      "content_chunk": {
        "type": "text",
        "analyzer": "ik_max_word"
      },

      "embedding": {
        "type": "knn_vector",
        "dimension": 768
      }
    }
  }
}
```

数据构建流程（Pipeline）

1. chunk切分
   chunk1: 0-300
   chunk2: 250-550
   chunk3: 500-800

2. 生成 embedding
   embedding = embedding(title + content_chunk)
3. 写入数据

```
POST news_index/_doc/123
{
  "id": "123",
  "title": "美联储加息影响市场",
  "content": "完整正文...",
  "publish_time": "2023-01-03T13:26:00",
  "source": "web_finance_news_db_20260107"
}

POST _bulk
{ "index": { "_index": "news_chunk_index", "_id": "123_1" } }
{ "doc_id": "123", "chunk_id": "123_1", "content_chunk": "...", "embedding": [...] }
{ "index": { "_index": "news_chunk_index", "_id": "123_2" } }
{ "doc_id": "123", "chunk_id": "123_2", "content_chunk": "...", "embedding": [...] }
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
podman exec -it opensearch-node1 bash

./bin/opensearch-plugin install \
https://get.infini.cloud/opensearch/analysis-ik/3.5.0

podman restart opensearch-node1

curl -X GET "localhost:9200/_analyze" -H 'Content-Type: application/json' -d'
{
  "analyzer": "ik_smart",
  "text": "美联储加息对市场的影响"
}'
```

 

```shell
curl -X PUT "http://localhost:9200/news_index" \
-H "Content-Type: application/json" \
-d '{
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
      "id": { 
        "type": "keyword" 
      },

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

      "publish_time": { 
        "type": "date" 
      },

      "source": { 
        "type": "keyword" 
      },

      "create_time": {
        "type": "date"
      }
    }
  }
}'

curl -X PUT "http://localhost:9200/news_chunk_index" \
-H "Content-Type: application/json" \
-d '{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "refresh_interval": "30s",
    "index": {
      "knn": true
    }
  },
  "mappings": {
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
            "ef_construction": 128,
            "m": 24
          }
        }
      }
    }
  }
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



```
{
    "code": 200,
    "data": [
        {
            "doc_id": 96312,
            "title": "中电港：与公司合作的小米包括北京小米电子产品有限公司、小米通讯技术有限公司等",
            "snippet": " 51019002002026号\n新闻职业道德监督热线：400 889 0008     邮箱：zbb@nbd.com.cn",
            "highlight": "中电港：与公司合作的<em>小米</em>包括北京<em>小米</em>电子产品有限公司、<em>小米</em>通讯技术有限公司等\n其儿媳此前发声明，双方矛盾持续升级\n6\n马杜罗预计5日在纽约“首次出庭”；元旦假期国内出游人次超1.4亿；茅台<em>集团</em>：将积极配合有关部门调查处理；宇树科技辟谣丨每经早参\n7\n特朗普：委内瑞拉可能不会是美国干预的最后一个国家，“我们绝对需要格陵兰岛”！\n 51019002002026号\n新闻职业道德监督热线：400 889 0008     邮箱：zbb@nbd.com.cn",
            "publish_time": "2023-10-27T16:09:30+00:00",
            "source": "web_finance_news_db_20260105"
        },
        {
            "doc_id": 100702,
            "title": "",
            "snippet": "年增100%。累计今年1至7月上市新机型238款，按年增6.7%，其中5G手机103款，按年减14.9%。\n郭家耀：小米软件服务收入占比有待提升 或可上试13元惟难大升\n汇盈国际资产管理董事总经理郭家耀指，小米有新手机推出是好事，近期智能手机市场随着有更多新机推出也略见回暖，5G手机出货量亦见好转。不过，苹果公司也不是单靠销售手机等硬件业务，小米在软件服务包括云端及互联网等收入占比有待提升，市场亦对其家电产品生态圈有期望。至于汽车业务，由于市场上有太多竞争者，市场对小米汽车的反应暂无从评估，亦难给予估值。股价方面，小米或可持续上升，甚至可上试13元，惟因市况欠佳，难期望太高。\n另值得一提，华为",
            "highlight": "年增100%。累计今年1至7月上市新机型238款，按年增6.7%，其中5G手机103款，按年减14.9%。\n郭家耀：小米软件服务收入占比有待提升 或可上试13元惟难大升\n汇盈国际资产管理董事总经理郭家耀指，小米有新手机推出是好事，近期智能手机市场随着有更多新机推出也略见回暖，5G手机出货量亦见好转。不过，苹果公司也不是单靠销售手机等硬件业务，小米在软件服务包括云端及互联网等收入占比有待提升，市场亦对其家电产品生态圈有期望。至于汽车业务，由于市场上有太多竞争者，市场对小米汽车的反应暂无从评估，亦难给予估值。股价方面，小米或可持续上升，甚至可上试13元，惟因市况欠佳，难期望太高。\n另值得一提，华为",
            "publish_time": "",
            "source": ""
        },
        {
            "doc_id": 79437,
            "title": "七年磨一剑，小米澎湃OS何以“济沧海”",
            "snippet": "康服务\n奇富科技发布首个信贷多模态评测基准，可全面评估信贷AI模型实战能力\n被小米辞退的王腾成立睡眠健康科技公司“今日宜休”\n雷军首度回应KOL接触事件：绝不容忍攻击诅咒米粉，保护用户是底线\n英伟达CEO黄仁勋盛赞中国工程师与AI研究者：属世界顶尖水平\n雷军：新一代SU7预计2026年4月起交付\n乘联会：2025年12月乘用车零售229.6万辆 新能源渗透率60.4%\n雷军详解新一代小米SU7全系标配激光雷达：可提升安全性\n雷军回应新一代SU7涨价：成本大涨下难实现加量不加价\n“好友太多会被封”？微信员工辟谣所谓“封号新规”\n更多\n苹果一名设计师跳槽AI初创公司 曾参与设计iPhone Air",
            "highlight": "七年磨一剑，<em>小米</em>澎湃OS何以“济沧海”\n在<em>小米</em>的价值排序中，第一自然是落在用户层，第二是合作伙伴和股东层，第三指向于社会价值层。对于<em>小米</em>而言，比起追求短期回报，更重要的是谋划未来，收入和利润也将是一个水到渠成的结果。\n<em>小米</em><em>集团</em>创始人、董事长兼CEO雷军表示，过去几年<em>小米</em>一直在突破认知，改变成长，今天将迎来“跨越时刻”。\n康服务\n奇富科技发布首个信贷多模态评测基准，可全面评估信贷AI模型实战能力\n被小米辞退的王腾成立睡眠健康科技公司“今日宜休”\n雷军首度回应KOL接触事件：绝不容忍攻击诅咒米粉，保护用户是底线\n英伟达CEO黄仁勋盛赞中国工程师与AI研究者：属世界顶尖水平\n雷军：新一代SU7预计2026年4月起交付\n乘联会：2025年12月乘用车零售229.6万辆 新能源渗透率60.4%\n雷军详解新一代小米SU7全系标配激光雷达：可提升安全性\n雷军回应新一代SU7涨价：成本大涨下难实现加量不加价\n“好友太多会被封”？微信员工辟谣所谓“封号新规”\n更多\n苹果一名设计师跳槽AI初创公司 曾参与设计iPhone Air",
            "publish_time": "2023-10-30T09:18:03+00:00",
            "source": "web_finance_news_db_20260108"
        },
        {
            "doc_id": 89607,
            "title": "同兴达：我司与小米有合作，目前正在积极拓展新客户",
            "snippet": "\n新闻职业道德监督热线：400 889 0008     邮箱：zbb@nbd.com.cn",
            "highlight": "同兴达：我司与<em>小米</em>有合作，目前正在积极拓展新客户\n他最新发文抱怨：挪威不把诺贝尔和平奖颁给我；他还与哥伦比亚总统首次通话，讨论了毒品问题\n2\n独家丨“收到钱了”，帮扶祥源控股<em>集团</em>工作组开启资金预清退，比例为投资本金5%，有人获退款10万余元\n3\n嘉必优大跌近13%\n4\n特朗普指示美国退出 “不符合该国利益”的66个 国际组织\n5\n陈志被押解回国\n6\n\n新闻职业道德监督热线：400 889 0008     邮箱：zbb@nbd.com.cn",
            "publish_time": "2023-10-30T15:32:31+00:00",
            "source": "web_finance_news_db_20260109"
        },
        {
            "doc_id": 92035,
            "title": "可立克：公司是小米的二级供应商",
            "snippet": "508号-3\n川公网安备 51019002002026号\n新闻职业道德监督热线：400 889 0008     邮箱：zbb@nbd.com.cn",
            "highlight": "可立克：公司是<em>小米</em>的二级供应商\n他最新发文抱怨：挪威不把诺贝尔和平奖颁给我；他还与哥伦比亚总统首次通话，讨论了毒品问题\n2\n嘉必优大跌近13%\n3\n独家丨“收到钱了”，帮扶祥源控股<em>集团</em>工作组开启资金预清退，比例为投资本金5%，有人获退款10万余元\n4\n特朗普指示美国退出 “不符合该国利益”的66个 国际组织\n5\n陈志被押解回国\n6\n508号-3\n川公网安备 51019002002026号\n新闻职业道德监督热线：400 889 0008     邮箱：zbb@nbd.com.cn",
            "publish_time": "2023-10-19T18:53:45+00:00",
            "source": "web_finance_news_db_20260109"
        },
        {
            "doc_id": 93358,
            "title": "盛帮股份：目前公司与小米无合作关系",
            "snippet": "加入我们\n每日经济新闻\n成都市天府科幻企业管理\n拟聘人员公示\nCopyright © 2025 每日经济新闻报社版权所有，未经许可不得转载使用，违者必究。\n广告热线  北京: 010-57613265， 上海: 021-61283008， 广州: 020-84201861， 深圳: 0755-83520159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 51019002002026号\n新闻职业道德监督热线：400 889 0008     邮箱：zbb@nbd.com.cn",
            "highlight": "盛帮股份：目前公司与<em>小米</em>无合作关系\n盛帮股份：目前公司与<em>小米</em>无合作关系 | 每经网\n每经网首页\n丨\n宏观\n丨\n金融\n丨\n公司\n丨\n视频\n丨\n券商\n丨\nIPO\n丨\n基金\n丨\n汽车\n丨\n房产\n丨\n新文化\n丨\n未来商业\n丨\n文创通\n丨\n城市\n丨\n每经商学院\n互动\n每经网首页\n>\n互动\n>\n        正文\n盛帮股份：目前公司与<em>小米</em>无合作关系\n加入我们\n每日经济新闻\n成都市天府科幻企业管理\n拟聘人员公示\nCopyright © 2025 每日经济新闻报社版权所有，未经许可不得转载使用，违者必究。\n广告热线  北京: 010-57613265， 上海: 021-61283008， 广州: 020-84201861， 深圳: 0755-83520159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 51019002002026号\n新闻职业道德监督热线：400 889 0008     邮箱：zbb@nbd.com.cn",
            "publish_time": "2023-10-30T16:10:27+00:00",
            "source": "web_finance_news_db_20251222"
        },
        {
            "doc_id": 83450,
            "title": "奕东电子：公司与小米有多年的良好合作关系，并持续保持良好的沟通与技术交流",
            "snippet": "服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 51019002002026号\n新闻职业道德监督热线：400 889 0008     邮箱：zbb@nbd.com.cn",
            "highlight": "奕东电子：公司与<em>小米</em>有多年的良好合作关系，并持续保持良好的沟通与技术交流\n，公司与<em>小米</em>有多年的良好合作关系，并持续保持良好的沟通与技术交流，公司会密切关注<em>小米</em>新能源汽车的发展情况，做好相关储备\n(记者 蔡鼎)\n免责声明：本文内容与数据仅供参考，不构成投资建议，使用前核实。\n服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 51019002002026号\n新闻职业道德监督热线：400 889 0008     邮箱：zbb@nbd.com.cn",
            "publish_time": "2023-10-20T10:18:03+00:00",
            "source": "web_finance_news_db_20251222"
        },
        {
            "doc_id": 82970,
            "title": "凯众股份：小米汽车目前尚未上市，公司的减振产品与小米汽车有同步开发业务",
            "snippet": "2\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 51019002002026号\n新闻职业道德监督热线：400 889 0008     邮箱：zbb@nbd.com.cn",
            "highlight": "凯众股份：<em>小米</em>汽车目前尚未上市，公司的减振产品与<em>小米</em>汽车有同步开发业务\n特朗普称美乌将达成安全协议；江西省博物馆声明：是原件；<em>小米</em><em>集团</em>林斌拟减持不超20亿美元；爱奇艺：安排退费丨每经早参\n2\n贵金属突然巨震\n3\n独家对话 | 王佶的“数据信仰”：从汽配到游戏、搏出千亿元市值，世纪华通如何用算法跑赢巨头？\n2\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 51019002002026号\n新闻职业道德监督热线：400 889 0008     邮箱：zbb@nbd.com.cn",
            "publish_time": "2023-10-11T16:36:09+00:00",
            "source": "web_finance_news_db_20251230"
        },
        {
            "doc_id": 97682,
            "title": "天奇股份：公司与小米汽车前期已有业务接洽",
            "snippet": "储和生产\n相关文章\n溯联股份：公司暂未与小米汽车有直接配套业务\n亚通精工：公司与小米汽车正在业务接洽中，争取相关合作\n玲珑轮胎：公司与小米汽车正在积极接洽，但目前暂未开展相关合作\n维科精密：公司目前与小米、问界暂无直接合作\n热文精选\n增速引领经济大市，成都2026年如何继续“挑大梁”？\n“AI PC”加速到来，哪些产业将被重塑？\n成都都市圈东进长三角的“三维”考量\n成都都市圈携手长三角 共拓低空经济新蓝海\n中西部经济第一大省，如何“开局”？\n点击排行\n1\n特朗普又威胁哥伦比亚总统佩特罗：他当不了太久总统，对哥伦比亚发动行动“听起来不错”！佩特罗计划今年继续对特朗普采取强硬立场\n2\n马杜罗预计5",
            "highlight": "天奇股份：公司与<em>小米</em>汽车前期已有业务接洽\n佩特罗计划今年继续对特朗普采取强硬立场\n2\n马杜罗预计5日在纽约“首次出庭”；元旦假期国内出游人次超1.4亿；茅台<em>集团</em>：将积极配合有关部门调查处理；宇树科技辟谣丨每经早参\n3\n特朗普：委内瑞拉可能不会是美国干预的最后一个国家，“我们绝对需要格陵兰岛”！\n储和生产\n相关文章\n溯联股份：公司暂未与小米汽车有直接配套业务\n亚通精工：公司与小米汽车正在业务接洽中，争取相关合作\n玲珑轮胎：公司与小米汽车正在积极接洽，但目前暂未开展相关合作\n维科精密：公司目前与小米、问界暂无直接合作\n热文精选\n增速引领经济大市，成都2026年如何继续“挑大梁”？\n“AI PC”加速到来，哪些产业将被重塑？\n成都都市圈东进长三角的“三维”考量\n成都都市圈携手长三角 共拓低空经济新蓝海\n中西部经济第一大省，如何“开局”？\n点击排行\n1\n特朗普又威胁哥伦比亚总统佩特罗：他当不了太久总统，对哥伦比亚发动行动“听起来不错”！佩特罗计划今年继续对特朗普采取强硬立场\n2\n马杜罗预计5",
            "publish_time": "2023-10-23T18:18:30+00:00",
            "source": "web_finance_news_db_20260106"
        },
        {
            "doc_id": 82606,
            "title": "中京电子：公司通过ODM厂商为小米等智能手机提供配套产品",
            "snippet": "案号：蜀ICP备19004508号-3\n川公网安备 51019002002026号\n新闻职业道德监督热线：400 889 0008     邮箱：zbb@nbd.com.cn",
            "highlight": "中京电子：公司通过ODM厂商为<em>小米</em>等智能手机提供配套产品\n公司通过ODM厂商为<em>小米</em>等智能手机提供配套产品\n每日经济新闻\n2023-10-30 10:23:02\n每经AI快讯，有投资者在投资者互动平台提问：贵公司给<em>小米</em>14提供什么电子产品\n中京电子（002579.SZ）10月30日在投资者互动平台表示，公司通过ODM厂商为<em>小米</em>等智能手机提供配套产品。\n案号：蜀ICP备19004508号-3\n川公网安备 51019002002026号\n新闻职业道德监督热线：400 889 0008     邮箱：zbb@nbd.com.cn",
            "publish_time": "2023-10-30T10:23:02+00:00",
            "source": "web_finance_news_db_20260108"
        },
        {
            "doc_id": 82813,
            "title": "一汽富维：未获得小米汽车量产业务",
            "snippet": "1019002002026号\n新闻职业道德监督热线：400 889 0008     邮箱：zbb@nbd.com.cn",
            "highlight": "一汽富维：未获得<em>小米</em>汽车量产业务\n一汽富维：公司已进入到<em>小米</em>供应商体系 尚未获得<em>小米</em>汽车零部件订单\n热文精选\n中西部经济第一大省，如何“开局”？\n1019002002026号\n新闻职业道德监督热线：400 889 0008     邮箱：zbb@nbd.com.cn",
            "publish_time": "2023-10-26T16:16:41+00:00",
            "source": "web_finance_news_db_20251226"
        },
        {
            "doc_id": 85255,
            "title": "集泰股份：公司产品暂未供应小米汽车",
            "snippet": "  北京: 010-57613265， 上海: 021-61283008， 广州: 020-84201861， 深圳: 0755-83520159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 51019002002026号\n新闻职业道德监督热线：400 889 0008     邮箱：zbb@nbd.com.cn",
            "highlight": "集泰股份：公司产品暂未供应<em>小米</em>汽车\n集泰股份：公司产品暂未供应<em>小米</em>汽车 | 每经网\n每经网首页\n丨\n宏观\n丨\n金融\n丨\n公司\n丨\n视频\n丨\n券商\n丨\nIPO\n丨\n基金\n丨\n汽车\n丨\n房产\n丨\n新文化\n丨\n未来商业\n丨\n文创通\n丨\n城市\n丨\n每经商学院\n互动\n每经网首页\n>\n互动\n>\n        正文\n集泰股份：公司产品暂未供应<em>小米</em>汽车\n  北京: 010-57613265， 上海: 021-61283008， 广州: 020-84201861， 深圳: 0755-83520159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 51019002002026号\n新闻职业道德监督热线：400 889 0008     邮箱：zbb@nbd.com.cn",
            "publish_time": "2023-10-17T15:30:34+00:00",
            "source": "web_finance_news_db_20260109"
        },
        {
            "doc_id": 95444,
            "title": "九安医疗：公司持有小米公司股票，未直接参与小米汽车业务",
            "snippet": "证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 51019002002026号\n新闻职业道德监督热线：400 889 0008     邮箱：zbb@nbd.com.cn",
            "highlight": "九安医疗：公司持有<em>小米</em>公司股票，未直接参与<em>小米</em>汽车业务\n上一篇文章\n英特<em>集团</em>：第三季度“英特转债”转股约1697万股\n返回每经网首页\n下一篇文章\n公司在新疆的业务拓展进展是否顺利？喜悦智行：公司相关业务目前处于市场调研和开拓阶段\n相关文章\nST曙光：<em>小米</em>公司、华为公司、蔚来汽车从未与公司进行任何形式的接触\n小爱同学，你闯祸啦！\n证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 51019002002026号\n新闻职业道德监督热线：400 889 0008     邮箱：zbb@nbd.com.cn",
            "publish_time": "2023-10-09T16:23:14+00:00",
            "source": "web_finance_news_db_20260107"
        },
        {
            "doc_id": 86190,
            "title": "浙江仙通：公司目前尚未与小米汽车开展合作",
            "snippet": " 020-84201861， 深圳: 0755-83520159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 51019002002026号\n新闻职业道德监督热线：400 889 0008     邮箱：zbb@nbd.com.cn",
            "highlight": "浙江仙通：公司目前尚未与<em>小米</em>汽车开展合作\n浙江仙通：公司目前尚未与<em>小米</em>汽车开展合作 | 每经网\n每经网首页\n丨\n宏观\n丨\n金融\n丨\n公司\n丨\n视频\n丨\n券商\n丨\nIPO\n丨\n基金\n丨\n汽车\n丨\n房产\n丨\n新文化\n丨\n未来商业\n丨\n文创通\n丨\n城市\n丨\n每经商学院\n互动\n每经网首页\n>\n互动\n>\n        正文\n浙江仙通：公司目前尚未与<em>小米</em>汽车开展合作\n 020-84201861， 深圳: 0755-83520159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 51019002002026号\n新闻职业道德监督热线：400 889 0008     邮箱：zbb@nbd.com.cn",
            "publish_time": "2023-10-17T00:51:28+00:00",
            "source": "web_finance_news_db_20251230"
        },
        {
            "doc_id": 98286,
            "title": "小米汽车生产调试冲刺 新“米链”公司透露合作进展",
            "snippet": "20159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 51019002002026号\n新闻职业道德监督热线：400 889 0008     邮箱：zbb@nbd.com.cn",
            "highlight": "<em>小米</em>汽车生产调试冲刺 新“米链”公司透露合作进展\n记者日前赴<em>小米</em>汽车通州生产基地实地调研获悉，这家新晋造车巨头正悄然进入紧张的生产调试冲刺阶段，<em>小米</em><em>集团</em>董事长雷军近期已带领<em>小米</em>汽车高层在新疆完成夏季新车路测，以争取在获得相关批文后尽快进入新车量产。\n20159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 51019002002026号\n新闻职业道德监督热线：400 889 0008     邮箱：zbb@nbd.com.cn",
            "publish_time": "2023-09-12T09:46:00+00:00",
            "source": "web_finance_news_db_20251230"
        },
        {
            "doc_id": 96529,
            "title": "思林杰：暂未与小米有订单合作",
            "snippet": "512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 51019002002026号\n新闻职业道德监督热线：400 889 0008     邮箱：zbb@nbd.com.cn",
            "highlight": "思林杰：暂未与<em>小米</em>有订单合作\n10月27日沪深股通净流入46.68亿，其中5.270亿买了它\n相关文章\n思林杰：公司与客户保持良好的合作关系\n光庭信息：公司目前暂未与Vinfast有合作\n锐明技术：公司暂未和<em>小米</em>有业务合作\n星星科技：目前部分零部件产品与<em>小米</em>有合作，但合作暂未涉及<em>小米</em>今年发布的新品项目\n热文精选\n增速引领经济大市，\n512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 51019002002026号\n新闻职业道德监督热线：400 889 0008     邮箱：zbb@nbd.com.cn",
            "publish_time": "2023-10-27T17:34:45+00:00",
            "source": "web_finance_news_db_20260101"
        },
        {
            "doc_id": 84246,
            "title": "天汽模：截至目前，与小米尚未有合作订单",
            "snippet": "关信息\n关于我们\n版权声明\n联系我们\n广告投放\n友情链接\n关注我们\n辟谣专区\n加入我们\n每日经济新闻\n成都市天府科幻企业管理\n拟聘人员公示\nCopyright © 2026 每日经济新闻报社版权所有，未经许可不得转载使用，违者必究。\n广告热线  北京: 010-57613265， 上海: 021-61283008， 广州: 020-84201861， 深圳: 0755-83520159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 51019002002026号\n新闻职业道德监督热线：400 88",
            "highlight": "天汽模：截至目前，与<em>小米</em>尚未有合作订单\n每日经济新闻\n2023-10-09 15:49:42\n每经AI快讯，有投资者在投资者互动平台提问：<em>小米</em>汽车预计明年上半年量产，作为合作伙伴，请问贵公司是否会积极对接<em>小米</em>，争取订单落地？\n关信息\n关于我们\n版权声明\n联系我们\n广告投放\n友情链接\n关注我们\n辟谣专区\n加入我们\n每日经济新闻\n成都市天府科幻企业管理\n拟聘人员公示\nCopyright © 2026 每日经济新闻报社版权所有，未经许可不得转载使用，违者必究。\n广告热线  北京: 010-57613265， 上海: 021-61283008， 广州: 020-84201861， 深圳: 0755-83520159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 51019002002026号\n新闻职业道德监督热线：400 88",
            "publish_time": "2023-10-09T15:49:42+00:00",
            "source": "web_finance_news_db_20260107"
        },
        {
            "doc_id": 90224,
            "title": "东箭科技：全资子公司维杰汽车与小米汽车也有技术交流",
            "snippet": "89 0008     邮箱：zbb@nbd.com.cn",
            "highlight": "东箭科技：全资子公司维杰汽车与<em>小米</em>汽车也有技术交流\n每经AI快讯，有投资者在投资者互动平台提问：公司与<em>小米</em>汽车有合作吗？\n89 0008     邮箱：zbb@nbd.com.cn",
            "publish_time": "2023-10-27T15:26:49+00:00",
            "source": "web_finance_news_db_20251222"
        },
        {
            "doc_id": 81381,
            "title": "一汽富维：公司当前没有和小米汽车开展同步开发业务",
            "snippet": "美国！\n欢迎关注每日经济新闻APP\n0\n0\n相关信息\n关于我们\n版权声明\n联系我们\n广告投放\n友情链接\n关注我们\n辟谣专区\n加入我们\n每日经济新闻\n成都市天府科幻企业管理\n拟聘人员公示\nCopyright © 2025 每日经济新闻报社版权所有，未经许可不得转载使用，违者必究。\n广告热线  北京: 010-57613265， 上海: 021-61283008， 广州: 020-84201861， 深圳: 0755-83520159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 5101900200",
            "highlight": "一汽富维：公司当前没有和<em>小米</em>汽车开展同步开发业务\n上一篇文章\n翔腾新材：公司与欧菲光暂无业务往来\n返回每经网首页\n下一篇文章\n新三板基础层公司农匠科技发生3笔大宗交易，总成交金额1155万元\n相关文章\n一汽富维：多家分子公司进入到<em>小米</em>汽车的采购组当中 <em>小米</em>第一款车型报价完毕\n一汽富维：公司正在积极与<em>小米</em>汽车进行业务沟通，多家分子公司已经进入到<em>小米</em>汽车的采购组当中\n美国！\n欢迎关注每日经济新闻APP\n0\n0\n相关信息\n关于我们\n版权声明\n联系我们\n广告投放\n友情链接\n关注我们\n辟谣专区\n加入我们\n每日经济新闻\n成都市天府科幻企业管理\n拟聘人员公示\nCopyright © 2025 每日经济新闻报社版权所有，未经许可不得转载使用，违者必究。\n广告热线  北京: 010-57613265， 上海: 021-61283008， 广州: 020-84201861， 深圳: 0755-83520159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 5101900200",
            "publish_time": "2023-10-17T15:58:19+00:00",
            "source": "web_finance_news_db_20251226"
        },
        {
            "doc_id": 87624,
            "title": "增程的香，小米也想尝尝",
            "snippet": "增程的香，<em>小米</em>也想尝尝\n微信员工辟谣所谓“封号新规”\n更多\n苹果一名设计师跳槽AI初创公司 曾参与设计iPhone Air\nAI初创公司Anthropic拟以3500亿美元估值 融资100亿美元\n电池制造商SK On的CEO在CES 2026期间 有参观吉利和现代汽车展台\n现代汽车<em>集团</em>会长郑义宣现身CES 2026 已同黄仁勋有过会面",
            "highlight": "增程的香，<em>小米</em>也想尝尝\n微信员工辟谣所谓“封号新规”\n更多\n苹果一名设计师跳槽AI初创公司 曾参与设计iPhone Air\nAI初创公司Anthropic拟以3500亿美元估值 融资100亿美元\n电池制造商SK On的CEO在CES 2026期间 有参观吉利和现代汽车展台\n现代汽车<em>集团</em>会长郑义宣现身CES 2026 已同黄仁勋有过会面",
            "publish_time": "2023-10-11T09:09:12+00:00",
            "source": "web_finance_news_db_20260108"
        },
        {
            "doc_id": 96514,
            "title": "德迈仕：公司产品不直接供应给小米汽车，公司生产的1 款方向丝杠最终用于小米汽车中",
            "snippet": "德迈仕：公司产品不直接供应给<em>小米</em>汽车，公司生产的1 款方向丝杠最终用于<em>小米</em>汽车中\n上一篇文章\n周大福：近3个月<em>集团</em>零售值同比增5.8%\n返回每经网首页\n下一篇文章\n注意！",
            "highlight": "德迈仕：公司产品不直接供应给<em>小米</em>汽车，公司生产的1 款方向丝杠最终用于<em>小米</em>汽车中\n上一篇文章\n周大福：近3个月<em>集团</em>零售值同比增5.8%\n返回每经网首页\n下一篇文章\n注意！",
            "publish_time": "2023-10-12T17:06:08+00:00",
            "source": "web_finance_news_db_20251222"
        },
        {
            "doc_id": 94765,
            "title": "今飞凯达：公司未向小米提供产品",
            "snippet": "今飞凯达：公司未向<em>小米</em>提供产品\n他最新发文抱怨：挪威不把诺贝尔和平奖颁给我；他还与哥伦比亚总统首次通话，讨论了毒品问题\n3\n独家丨“收到钱了”，帮扶祥源控股<em>集团</em>工作组开启资金预清退，比例为投资本金5%，有人获退款10万余元\n4\n嘉必优大跌近13%\n5\n陈志被押解回国\n6\n郁亮辞职\n7\n中韩经贸合作呈现哪些新特点？",
            "highlight": "今飞凯达：公司未向<em>小米</em>提供产品\n他最新发文抱怨：挪威不把诺贝尔和平奖颁给我；他还与哥伦比亚总统首次通话，讨论了毒品问题\n3\n独家丨“收到钱了”，帮扶祥源控股<em>集团</em>工作组开启资金预清退，比例为投资本金5%，有人获退款10万余元\n4\n嘉必优大跌近13%\n5\n陈志被押解回国\n6\n郁亮辞职\n7\n中韩经贸合作呈现哪些新特点？",
            "publish_time": "2023-10-27T16:18:25+00:00",
            "source": "web_finance_news_db_20260109"
        },
        {
            "doc_id": 95324,
            "title": "",
            "snippet": "，南方总部即将成立\n10\n小米最新通报\n欢迎关注每日经济新闻APP\n0\n0\n相关信息\n关于我们\n版权声明\n联系我们\n广告投放\n友情链接\n关注我们\n辟谣专区\n加入我们\n每日经济新闻\n成都市天府科幻企业管理\n拟聘人员公示\nCopyright © 2026 每日经济新闻报社版权所有，未经许可不得转载使用，违者必究。\n广告热线  北京: 010-57613265， 上海: 021-61283008， 广州: 020-84201861， 深圳: 0755-83520159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n",
            "highlight": "，南方总部即将成立\n10\n小米最新通报\n欢迎关注每日经济新闻APP\n0\n0\n相关信息\n关于我们\n版权声明\n联系我们\n广告投放\n友情链接\n关注我们\n辟谣专区\n加入我们\n每日经济新闻\n成都市天府科幻企业管理\n拟聘人员公示\nCopyright © 2026 每日经济新闻报社版权所有，未经许可不得转载使用，违者必究。\n广告热线  北京: 010-57613265， 上海: 021-61283008， 广州: 020-84201861， 深圳: 0755-83520159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n",
            "publish_time": "",
            "source": ""
        },
        {
            "doc_id": 88405,
            "title": "德迈仕：公司产品不直接供应给小米汽车，公司生产的1款方向丝杠最终用于小米汽车中",
            "snippet": "德迈仕：公司产品不直接供应给<em>小米</em>汽车，公司生产的1款方向丝杠最终用于<em>小米</em>汽车中\n正文\n德迈仕：公司产品不直接供应给<em>小米</em>汽车，公司生产的1款方向丝杠最终用于<em>小米</em>汽车中\n每日经济新闻\n2023-10-31 15:09:09\n每经AI快讯，有投资者在投资者互动平台提问：公司间接供应给<em>小米</em>汽车哪些零部件？",
            "highlight": "德迈仕：公司产品不直接供应给<em>小米</em>汽车，公司生产的1款方向丝杠最终用于<em>小米</em>汽车中\n正文\n德迈仕：公司产品不直接供应给<em>小米</em>汽车，公司生产的1款方向丝杠最终用于<em>小米</em>汽车中\n每日经济新闻\n2023-10-31 15:09:09\n每经AI快讯，有投资者在投资者互动平台提问：公司间接供应给<em>小米</em>汽车哪些零部件？",
            "publish_time": "2023-10-31T15:09:09+00:00",
            "source": "web_finance_news_db_20251222"
        },
        {
            "doc_id": 85888,
            "title": "",
            "snippet": "成立\n10\n小米最新通报\n欢迎关注每日经济新闻APP\n0\n0\n相关信息\n关于我们\n版权声明\n联系我们\n广告投放\n友情链接\n关注我们\n辟谣专区\n加入我们\n每日经济新闻\n成都市天府科幻企业管理\n拟聘人员公示\nCopyright © 2026 每日经济新闻报社版权所有，未经许可不得转载使用，违者必究。\n广告热线  北京: 010-57613265， 上海: 021-61283008， 广州: 020-84201861， 深圳: 0755-83520159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 5",
            "highlight": "成立\n10\n小米最新通报\n欢迎关注每日经济新闻APP\n0\n0\n相关信息\n关于我们\n版权声明\n联系我们\n广告投放\n友情链接\n关注我们\n辟谣专区\n加入我们\n每日经济新闻\n成都市天府科幻企业管理\n拟聘人员公示\nCopyright © 2026 每日经济新闻报社版权所有，未经许可不得转载使用，违者必究。\n广告热线  北京: 010-57613265， 上海: 021-61283008， 广州: 020-84201861， 深圳: 0755-83520159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 5",
            "publish_time": "",
            "source": ""
        },
        {
            "doc_id": 79519,
            "title": "原创苹果第一，华为第二，小米第三，这榜单你们服吗？",
            "snippet": "原创苹果第一，华为第二，<em>小米</em>第三，这榜单你们服吗？\n在此必须表扬一下<em>小米</em>，要知道这个品牌曾经四年间不做平板。到了2022年才推出<em>小米</em>平板5系列，一经推出就大受欢迎，今年的<em>小米</em>平板6系列也是如此。目前在国内仅用一年的时间就做到前三，这说明了什么呢？",
            "highlight": "原创苹果第一，华为第二，<em>小米</em>第三，这榜单你们服吗？\n在此必须表扬一下<em>小米</em>，要知道这个品牌曾经四年间不做平板。到了2022年才推出<em>小米</em>平板5系列，一经推出就大受欢迎，今年的<em>小米</em>平板6系列也是如此。目前在国内仅用一年的时间就做到前三，这说明了什么呢？",
            "publish_time": "2023-10-08T07:08:09+00:00",
            "source": "web_finance_news_db_20260106"
        },
        {
            "doc_id": 92933,
            "title": "东箭科技：全资子公司维杰汽车与小米汽车有技术交流",
            "snippet": "东箭科技：全资子公司维杰汽车与<em>小米</em>汽车有技术交流\n每经AI快讯，有投资者在投资者互动平台提问：尊敬的董秘您好，请问公司与<em>小米</em>汽车是否有合作或正在洽谈合作，谢谢。",
            "highlight": "东箭科技：全资子公司维杰汽车与<em>小米</em>汽车有技术交流\n每经AI快讯，有投资者在投资者互动平台提问：尊敬的董秘您好，请问公司与<em>小米</em>汽车是否有合作或正在洽谈合作，谢谢。",
            "publish_time": "2023-10-13T16:37:30+00:00",
            "source": "web_finance_news_db_20251222"
        },
        {
            "doc_id": 86634,
            "title": "公司与小米汽车有合作开发吗？隆基机械：目前尚未正式合作",
            "snippet": "公司与<em>小米</em>汽车有合作开发吗？隆基机械：目前尚未正式合作\n公司与<em>小米</em>汽车有合作开发吗？",
            "highlight": "公司与<em>小米</em>汽车有合作开发吗？隆基机械：目前尚未正式合作\n公司与<em>小米</em>汽车有合作开发吗？",
            "publish_time": "2023-10-26T15:27:47+00:00",
            "source": "web_finance_news_db_20260107"
        },
        {
            "doc_id": 90467,
            "title": "合兴股份：公司目前供货产品暂未涉及问界、小米等系列车型",
            "snippet": "合兴股份：公司目前供货产品暂未涉及问界、<em>小米</em>等系列车型\n、<em>小米</em>等系列车型\n每日经济新闻\n2023-10-31 16:15:07\n每经AI快讯，有投资者在投资者互动平台提问：您好，请问公司有进入问界系列，和<em>小米</em>汽车供应链吗？",
            "highlight": "合兴股份：公司目前供货产品暂未涉及问界、<em>小米</em>等系列车型\n、<em>小米</em>等系列车型\n每日经济新闻\n2023-10-31 16:15:07\n每经AI快讯，有投资者在投资者互动平台提问：您好，请问公司有进入问界系列，和<em>小米</em>汽车供应链吗？",
            "publish_time": "2023-10-31T16:15:07+00:00",
            "source": "web_finance_news_db_20251231"
        },
        {
            "doc_id": 101102,
            "title": "玲珑轮胎：公司与小米汽车正在积极接洽，但目前暂未开展相关合作",
            "snippet": "玲珑轮胎：公司与<em>小米</em>汽车正在积极接洽，但目前暂未开展相关合作\n特朗普称美乌将达成安全协议；江西省博物馆声明：是原件；<em>小米</em><em>集团</em>林斌拟减持不超20亿美元；爱奇艺：安排退费丨每经早参\n4\n贵金属突然巨震\n5\n上海独居女子离世引关注，超百万元房产无人继承， 记者实探→\n6\n乌克兰袭击莫斯科\n7\n韩总统府迁回青瓦台\n8\n净资产仅剩4300多万元！",
            "highlight": "玲珑轮胎：公司与<em>小米</em>汽车正在积极接洽，但目前暂未开展相关合作\n特朗普称美乌将达成安全协议；江西省博物馆声明：是原件；<em>小米</em><em>集团</em>林斌拟减持不超20亿美元；爱奇艺：安排退费丨每经早参\n4\n贵金属突然巨震\n5\n上海独居女子离世引关注，超百万元房产无人继承， 记者实探→\n6\n乌克兰袭击莫斯科\n7\n韩总统府迁回青瓦台\n8\n净资产仅剩4300多万元！",
            "publish_time": "2023-09-19T10:51:20+00:00",
            "source": "web_finance_news_db_20251229"
        },
        {
            "doc_id": 94594,
            "title": "",
            "snippet": "成立\n10\n小米最新通报\n欢迎关注每日经济新闻APP\n0\n0\n相关信息\n关于我们\n版权声明\n联系我们\n广告投放\n友情链接\n关注我们\n辟谣专区\n加入我们\n每日经济新闻\n成都市天府科幻企业管理\n拟聘人员公示\nCopyright © 2026 每日经济新闻报社版权所有，未经许可不得转载使用，违者必究。\n广告热线  北京: 010-57613265， 上海: 021-61283008， 广州: 020-84201861， 深圳: 0755-83520159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 5",
            "highlight": "成立\n10\n小米最新通报\n欢迎关注每日经济新闻APP\n0\n0\n相关信息\n关于我们\n版权声明\n联系我们\n广告投放\n友情链接\n关注我们\n辟谣专区\n加入我们\n每日经济新闻\n成都市天府科幻企业管理\n拟聘人员公示\nCopyright © 2026 每日经济新闻报社版权所有，未经许可不得转载使用，违者必究。\n广告热线  北京: 010-57613265， 上海: 021-61283008， 广州: 020-84201861， 深圳: 0755-83520159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 5",
            "publish_time": "",
            "source": ""
        },
        {
            "doc_id": 100589,
            "title": "罗永浩“间接现身”Redmi发布会，曾大赞小米消灭山寨厂功德无量",
            "snippet": "罗永浩“间接现身”Redmi发布会，曾大赞<em>小米</em>消灭山寨厂功德无量\n这是典型的良币驱逐劣币，从这个意义上看，<em>小米</em>功德无量！\n”\nIT之家此前报道，今年 7 月 31 日<em>小米</em><em>集团</em>公关部总经理 @王化 发文表示，十年前的今天，<em>小米</em><em>集团</em> CEO 雷军和中国移动高管现场宣布红米手机 1 代正式诞生。",
            "highlight": "罗永浩“间接现身”Redmi发布会，曾大赞<em>小米</em>消灭山寨厂功德无量\n这是典型的良币驱逐劣币，从这个意义上看，<em>小米</em>功德无量！\n”\nIT之家此前报道，今年 7 月 31 日<em>小米</em><em>集团</em>公关部总经理 @王化 发文表示，十年前的今天，<em>小米</em><em>集团</em> CEO 雷军和中国移动高管现场宣布红米手机 1 代正式诞生。",
            "publish_time": "2023-09-21T21:42:37+00:00",
            "source": "web_finance_news_db_20251227"
        },
        {
            "doc_id": 89716,
            "title": "视源股份：公司正在为小米电视 S Pro 65/75 英寸版系列产品提供主控板卡",
            "snippet": "视源股份：公司正在为<em>小米</em>电视 S Pro 65/75 英寸版系列产品提供主控板卡\n正文\n视源股份：公司正在为<em>小米</em>电视 S Pro 65/75 英寸版系列产品提供主控板卡\n每日经济新闻\n2023-10-25 08:16:09\n每经AI快讯，有投资者在投资者互动平台提问：近期<em>小米</em>电视s pro 65和75预售期间就卖断货，是否采用公司显示主板。",
            "highlight": "视源股份：公司正在为<em>小米</em>电视 S Pro 65/75 英寸版系列产品提供主控板卡\n正文\n视源股份：公司正在为<em>小米</em>电视 S Pro 65/75 英寸版系列产品提供主控板卡\n每日经济新闻\n2023-10-25 08:16:09\n每经AI快讯，有投资者在投资者互动平台提问：近期<em>小米</em>电视s pro 65和75预售期间就卖断货，是否采用公司显示主板。",
            "publish_time": "2023-10-25T08:16:09+00:00",
            "source": "web_finance_news_db_20260109"
        },
        {
            "doc_id": 84406,
            "title": "这么多年竟没发现 华为、小米手机集成了迅雷下载 咋做到的--快科技--科技改变未来",
            "snippet": "这么多年竟没发现 华为、<em>小米</em>手机集成了迅雷下载 咋做到的--快科技--科技改变未来\n【本文结束】如需转载请务必注明出处：快科技\n责任编辑：随心\n相关资讯\n<em>小米</em>14最后一款配色公布 雷军：最经典最深邃的黑\n雷军评价<em>小米</em>14屏幕：参数非常夸张 超级省电 特别恐怖\n<em>小米</em>14全新配色岩石青公布 雷军：非常喜欢 设计师下了很大功夫\n为何要自研澎湃OS、还是安卓吗、两者有何关系：<em>小米</em>卢伟冰释疑！",
            "highlight": "这么多年竟没发现 华为、<em>小米</em>手机集成了迅雷下载 咋做到的--快科技--科技改变未来\n【本文结束】如需转载请务必注明出处：快科技\n责任编辑：随心\n相关资讯\n<em>小米</em>14最后一款配色公布 雷军：最经典最深邃的黑\n雷军评价<em>小米</em>14屏幕：参数非常夸张 超级省电 特别恐怖\n<em>小米</em>14全新配色岩石青公布 雷军：非常喜欢 设计师下了很大功夫\n为何要自研澎湃OS、还是安卓吗、两者有何关系：<em>小米</em>卢伟冰释疑！",
            "publish_time": "2023-10-25T16:01:09+00:00",
            "source": "web_finance_news_db_20260107"
        },
        {
            "doc_id": 93342,
            "title": "",
            "snippet": "雷军：史上最好看小米影像旗舰\n华为Mate 70 Air 16GB内存版开售 采用麒麟9020A\n新能源汽车电池安全新要求强制要求不起火不爆炸\n全球销量前10电动汽车8款来自中国：小米等上榜\n关于本站\n管理团队\n版权申明\n网站地图\n联系合作\nWAP版\n移动客户端\n招聘信息\nCopyright © 2005-2020\nTechWeb.COM.CN\n-\ncandou.com\nAll rights reserved.\n京ICP证060517号/京ICP备09080754号\n京公网安备11010802020576号\nTechWeb公众号",
            "highlight": "雷军：史上最好看小米影像旗舰\n华为Mate 70 Air 16GB内存版开售 采用麒麟9020A\n新能源汽车电池安全新要求强制要求不起火不爆炸\n全球销量前10电动汽车8款来自中国：小米等上榜\n关于本站\n管理团队\n版权申明\n网站地图\n联系合作\nWAP版\n移动客户端\n招聘信息\nCopyright © 2005-2020\nTechWeb.COM.CN\n-\ncandou.com\nAll rights reserved.\n京ICP证060517号/京ICP备09080754号\n京公网安备11010802020576号\nTechWeb公众号",
            "publish_time": "",
            "source": ""
        },
        {
            "doc_id": 96396,
            "title": "大族激光：公司和小米、华为的合作主要集中在消费电子行业_证券要闻_顶尖财经网",
            "snippet": "大族激光：公司和<em>小米</em>、华为的合作主要集中在消费电子行业_证券要闻_顶尖财经网\n公司和<em>小米</em>、华为的合作主要集中在消费电子行业。\n编辑：\n来源：",
            "highlight": "大族激光：公司和<em>小米</em>、华为的合作主要集中在消费电子行业_证券要闻_顶尖财经网\n公司和<em>小米</em>、华为的合作主要集中在消费电子行业。\n编辑：\n来源：",
            "publish_time": "2023-10-12T17:51:31+00:00",
            "source": "web_finance_news_db_20260106"
        },
        {
            "doc_id": 98609,
            "title": "研讨农耕文化与产业创新 第十届世界小米起源与发展会议在敖汉举行",
            "snippet": "研讨农耕文化与产业创新 第十届世界<em>小米</em>起源与发展会议在敖汉举行\n第十届世界<em>小米</em>起源与发展会议由中国社会科学院考古研究所、中国作物学会粟类作物专业委员会、国家谷子高粱产业技术研发中心、中国国家品牌网、内蒙古自治区农牧厅、中共敖汉旗委、敖汉旗人民政府、敖汉旗龙兴国有资本运营(<em>集团</em>)有限责任公司共同主办，敖汉旗农耕<em>小米</em>产业发展(<em>集团</em>)有限公司承办。",
            "highlight": "研讨农耕文化与产业创新 第十届世界<em>小米</em>起源与发展会议在敖汉举行\n第十届世界<em>小米</em>起源与发展会议由中国社会科学院考古研究所、中国作物学会粟类作物专业委员会、国家谷子高粱产业技术研发中心、中国国家品牌网、内蒙古自治区农牧厅、中共敖汉旗委、敖汉旗人民政府、敖汉旗龙兴国有资本运营(<em>集团</em>)有限责任公司共同主办，敖汉旗农耕<em>小米</em>产业发展(<em>集团</em>)有限公司承办。",
            "publish_time": "2023-09-02T19:08:00+00:00",
            "source": "web_finance_news_db_20251224"
        },
        {
            "doc_id": 98644,
            "title": "",
            "snippet": "小米最新通报\n欢迎关注每日经济新闻APP\n0\n0\n相关信息\n关于我们\n版权声明\n联系我们\n广告投放\n友情链接\n关注我们\n辟谣专区\n加入我们\n每日经济新闻\n成都市天府科幻企业管理\n拟聘人员公示\nCopyright © 2026 每日经济新闻报社版权所有，未经许可不得转载使用，违者必究。\n广告热线  北京: 010-57613265， 上海: 021-61283008， 广州: 020-84201861， 深圳: 0755-83520159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 5101900",
            "highlight": "小米最新通报\n欢迎关注每日经济新闻APP\n0\n0\n相关信息\n关于我们\n版权声明\n联系我们\n广告投放\n友情链接\n关注我们\n辟谣专区\n加入我们\n每日经济新闻\n成都市天府科幻企业管理\n拟聘人员公示\nCopyright © 2026 每日经济新闻报社版权所有，未经许可不得转载使用，违者必究。\n广告热线  北京: 010-57613265， 上海: 021-61283008， 广州: 020-84201861， 深圳: 0755-83520159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 5101900",
            "publish_time": "",
            "source": ""
        },
        {
            "doc_id": 91300,
            "title": "甘肃电投集团与中能建建筑集团、甘肃电气集团签订战略合作协议",
            "snippet": "甘肃电投<em>集团</em>与中能建建筑<em>集团</em>、甘肃电气<em>集团</em>签订战略合作协议\n点击排行\n1\n成都农商银行资产规模突破万亿，农商行头部阵营形成“五强”格局\n2\n马杜罗夫妇熟睡中被拖走\n3\n雷军直播现场拆<em>小米</em>YU7：希望大家说公道话，不要故意找茬！",
            "highlight": "甘肃电投<em>集团</em>与中能建建筑<em>集团</em>、甘肃电气<em>集团</em>签订战略合作协议\n点击排行\n1\n成都农商银行资产规模突破万亿，农商行头部阵营形成“五强”格局\n2\n马杜罗夫妇熟睡中被拖走\n3\n雷军直播现场拆<em>小米</em>YU7：希望大家说公道话，不要故意找茬！",
            "publish_time": "2023-10-18T06:39:27+00:00",
            "source": "web_finance_news_db_20260104"
        },
        {
            "doc_id": 84419,
            "title": "汽车早知道丨理想汽车战略会将智驾提到空前高度，特斯拉起诉小米持股汽车产业链公司",
            "snippet": "汽车早知道丨理想汽车战略会将智驾提到空前高度，特斯拉起诉<em>小米</em>持股汽车产业链公司\n2\n、特斯拉（\nTSLA\n，股价：\n262.684\n美元，市值：\n8337.6\n亿美元）：\n10\n月\n11\n日，特斯拉针对此前起诉<em>小米</em>持股公司的事件做出最新回应。",
            "highlight": "汽车早知道丨理想汽车战略会将智驾提到空前高度，特斯拉起诉<em>小米</em>持股汽车产业链公司\n2\n、特斯拉（\nTSLA\n，股价：\n262.684\n美元，市值：\n8337.6\n亿美元）：\n10\n月\n11\n日，特斯拉针对此前起诉<em>小米</em>持股公司的事件做出最新回应。",
            "publish_time": "2023-10-12T08:47:40+00:00",
            "source": "web_finance_news_db_20260108"
        },
        {
            "doc_id": 90739,
            "title": "大族激光智能装备集团再次中标徐工集团千万级项目",
            "snippet": "大族激光智能装备<em>集团</em>再次中标徐工<em>集团</em>千万级项目\n2023-10-25 10:22:21\n每经AI快讯，据大族激光智能装备<em>集团</em>公众号，近日，大族激光智能装备<em>集团</em>再次中标徐工<em>集团</em>千万级光纤激光切割机及自动化生产线项目。",
            "highlight": "大族激光智能装备<em>集团</em>再次中标徐工<em>集团</em>千万级项目\n2023-10-25 10:22:21\n每经AI快讯，据大族激光智能装备<em>集团</em>公众号，近日，大族激光智能装备<em>集团</em>再次中标徐工<em>集团</em>千万级光纤激光切割机及自动化生产线项目。",
            "publish_time": "2023-10-25T10:22:21+00:00",
            "source": "web_finance_news_db_20260106"
        },
        {
            "doc_id": 97967,
            "title": "中央第十四巡视组进驻中航集团、东航集团、南航集团_证券要闻_顶尖财经网",
            "snippet": "中央第十四巡视组进驻中航<em>集团</em>、东航<em>集团</em>、南航<em>集团</em>_证券要闻_顶尖财经网\n中央第十四巡视组进驻中航<em>集团</em>、东航<em>集团</em>、南航<em>集团</em>_证券要闻_顶尖财经网\n您的位置：\n首页\n>>\n证券要闻\n>> 文章正文\n中央第十四巡视组进驻中航<em>集团</em>、东航<em>集团</em>、南航<em>集团</em>\n加入日期：2023-10-22 19:39:25\n顶尖财经网\n(www.58188.com)2023-10-22 19:39:25",
            "highlight": "中央第十四巡视组进驻中航<em>集团</em>、东航<em>集团</em>、南航<em>集团</em>_证券要闻_顶尖财经网\n中央第十四巡视组进驻中航<em>集团</em>、东航<em>集团</em>、南航<em>集团</em>_证券要闻_顶尖财经网\n您的位置：\n首页\n>>\n证券要闻\n>> 文章正文\n中央第十四巡视组进驻中航<em>集团</em>、东航<em>集团</em>、南航<em>集团</em>\n加入日期：2023-10-22 19:39:25\n顶尖财经网\n(www.58188.com)2023-10-22 19:39:25",
            "publish_time": "2023-10-22T19:39:25+00:00",
            "source": "web_finance_news_db_20251223"
        },
        {
            "doc_id": 90372,
            "title": "北汽集团与天津港集团签署战略合作框架协议",
            "snippet": "北汽<em>集团</em>与天津港<em>集团</em>签署战略合作框架协议\n2023-10-19 17:06:21\n每经AI快讯，据北汽<em>集团</em>消息，10月18日，北汽<em>集团</em>与天津港<em>集团</em>签署战略合作框架协议。",
            "highlight": "北汽<em>集团</em>与天津港<em>集团</em>签署战略合作框架协议\n2023-10-19 17:06:21\n每经AI快讯，据北汽<em>集团</em>消息，10月18日，北汽<em>集团</em>与天津港<em>集团</em>签署战略合作框架协议。",
            "publish_time": "2023-10-19T17:06:21+00:00",
            "source": "web_finance_news_db_20251226"
        },
        {
            "doc_id": 79399,
            "title": "",
            "snippet": "： Copyright © 1996-\n年",
            "highlight": "： Copyright © 1996-\n年",
            "publish_time": "",
            "source": ""
        },
        {
            "doc_id": 89872,
            "title": "中交一公局集团与陕煤集团签署战略合作协议",
            "snippet": "中交一公局<em>集团</em>与陕煤<em>集团</em>签署战略合作协议\n2023-10-22 08:10:28\n每经AI快讯，据中交一公局<em>集团</em>公众号21日消息，10月20日，中交一公局<em>集团</em>与陕西煤业化工<em>集团</em>有限责任公司在公司总部签署战略合作协议。",
            "highlight": "中交一公局<em>集团</em>与陕煤<em>集团</em>签署战略合作协议\n2023-10-22 08:10:28\n每经AI快讯，据中交一公局<em>集团</em>公众号21日消息，10月20日，中交一公局<em>集团</em>与陕西煤业化工<em>集团</em>有限责任公司在公司总部签署战略合作协议。",
            "publish_time": "2023-10-22T08:10:28+00:00",
            "source": "web_finance_news_db_20251228"
        },
        {
            "doc_id": 89391,
            "title": "",
            "snippet": "代小米SU7开启预售  雷军：预计2026年4月上市\n欢迎关注每日经济新闻APP\n0\n0\n相关信息\n关于我们\n版权声明\n联系我们\n广告投放\n友情链接\n关注我们\n辟谣专区\n加入我们\n每日经济新闻\n成都市天府科幻企业管理\n拟聘人员公示\nCopyright © 2026 每日经济新闻报社版权所有，未经许可不得转载使用，违者必究。\n广告热线  北京: 010-57613265， 上海: 021-61283008， 广州: 020-84201861， 深圳: 0755-83520159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004",
            "highlight": "代小米SU7开启预售  雷军：预计2026年4月上市\n欢迎关注每日经济新闻APP\n0\n0\n相关信息\n关于我们\n版权声明\n联系我们\n广告投放\n友情链接\n关注我们\n辟谣专区\n加入我们\n每日经济新闻\n成都市天府科幻企业管理\n拟聘人员公示\nCopyright © 2026 每日经济新闻报社版权所有，未经许可不得转载使用，违者必究。\n广告热线  北京: 010-57613265， 上海: 021-61283008， 广州: 020-84201861， 深圳: 0755-83520159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004",
            "publish_time": "",
            "source": ""
        },
        {
            "doc_id": 93151,
            "title": "押注碳汇？央企中林集团由三峡集团实施托管",
            "snippet": "央企中林<em>集团</em>由三峡<em>集团</em>实施托管\n央企中林<em>集团</em>由三峡<em>集团</em>实施托管\n2023-10-07 14:17:46\n来源：环保在线\n编辑：小易\n关键词：\n碳汇\n碳交易\n阅读量：35185\n导读：\n据悉，中林<em>集团</em>将由三峡<em>集团</em>实施托管，目前该公司的生产经营正常、债券兑付正常。",
            "highlight": "央企中林<em>集团</em>由三峡<em>集团</em>实施托管\n央企中林<em>集团</em>由三峡<em>集团</em>实施托管\n2023-10-07 14:17:46\n来源：环保在线\n编辑：小易\n关键词：\n碳汇\n碳交易\n阅读量：35185\n导读：\n据悉，中林<em>集团</em>将由三峡<em>集团</em>实施托管，目前该公司的生产经营正常、债券兑付正常。",
            "publish_time": "2023-10-07T14:17:46+00:00",
            "source": "web_finance_news_db_20260104"
        },
        {
            "doc_id": 94785,
            "title": "富奥股份：公司已获得问界减振器、圆簧和稳定杆产品订单，已获得小米紧固件和转向柱产品订单",
            "snippet": "富奥股份：公司已获得问界减振器、圆簧和稳定杆产品订单，已获得<em>小米</em>紧固件和转向柱产品订单\n公司已获得问界减振器、圆簧和稳定杆产品订单，已获得<em>小米</em>紧固件和转向柱产品订单。",
            "highlight": "富奥股份：公司已获得问界减振器、圆簧和稳定杆产品订单，已获得<em>小米</em>紧固件和转向柱产品订单\n公司已获得问界减振器、圆簧和稳定杆产品订单，已获得<em>小米</em>紧固件和转向柱产品订单。",
            "publish_time": "2023-10-24T18:00:19+00:00",
            "source": "web_finance_news_db_20260103"
        },
        {
            "doc_id": 82597,
            "title": "国泰集团： 获得政府补助",
            "snippet": "国泰<em>集团</em>： 获得政府补助\n上一篇文章\n德赛西威：本期计提信用与资产减值准备，影响2023年1-9月利润总额约1.69亿元\n返回每经网首页\n下一篇文章\n国泰<em>集团</em>：2023年前三季度净利润约2.26亿元，同比增加12.41%\n相关文章\n运机<em>集团</em>：获得政府补助\n立中<em>集团</em>：获得政府补助5090万元\n国泰<em>集团</em>：获得政府补助约457万元",
            "highlight": "国泰<em>集团</em>： 获得政府补助\n上一篇文章\n德赛西威：本期计提信用与资产减值准备，影响2023年1-9月利润总额约1.69亿元\n返回每经网首页\n下一篇文章\n国泰<em>集团</em>：2023年前三季度净利润约2.26亿元，同比增加12.41%\n相关文章\n运机<em>集团</em>：获得政府补助\n立中<em>集团</em>：获得政府补助5090万元\n国泰<em>集团</em>：获得政府补助约457万元",
            "publish_time": "2023-10-24T19:33:50+00:00",
            "source": "web_finance_news_db_20251231"
        },
        {
            "doc_id": 83418,
            "title": "",
            "snippet": "小米SU7开启预售  雷军：预计2026年4月上市\n欢迎关注每日经济新闻APP\n0\n0\n相关信息\n关于我们\n版权声明\n联系我们\n广告投放\n友情链接\n关注我们\n辟谣专区\n加入我们\n每日经济新闻\n成都市天府科幻企业管理\n拟聘人员公示\nCopyright © 2026 每日经济新闻报社版权所有，未经许可不得转载使用，违者必究。\n广告热线  北京: 010-57613265， 上海: 021-61283008， 广州: 020-84201861， 深圳: 0755-83520159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备190045",
            "highlight": "小米SU7开启预售  雷军：预计2026年4月上市\n欢迎关注每日经济新闻APP\n0\n0\n相关信息\n关于我们\n版权声明\n联系我们\n广告投放\n友情链接\n关注我们\n辟谣专区\n加入我们\n每日经济新闻\n成都市天府科幻企业管理\n拟聘人员公示\nCopyright © 2026 每日经济新闻报社版权所有，未经许可不得转载使用，违者必究。\n广告热线  北京: 010-57613265， 上海: 021-61283008， 广州: 020-84201861， 深圳: 0755-83520159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备190045",
            "publish_time": "",
            "source": ""
        },
        {
            "doc_id": 86103,
            "title": "安徽省生态环境集团与海螺集团签署战略合作协议",
            "snippet": "安徽省生态环境<em>集团</em>与海螺<em>集团</em>签署战略合作协议\n个人注册\n个人登录\n您的位置：\n环保\n固废处理\n水泥窑协同处置\n企业\n正文\n安徽省生态环境<em>集团</em>与海螺<em>集团</em>签署战略合作协议\n2023-10-26 14:57\n来源：\n安徽省生态环境<em>集团</em>\n关键词：\n生态环境<em>集团</em>\n海螺<em>集团</em>\n环保产业\n收藏\n点赞\n分享\n订阅\n投稿\n我要投稿\n10月24日，安徽省\n生态环境<em>集团</em>",
            "highlight": "安徽省生态环境<em>集团</em>与海螺<em>集团</em>签署战略合作协议\n个人注册\n个人登录\n您的位置：\n环保\n固废处理\n水泥窑协同处置\n企业\n正文\n安徽省生态环境<em>集团</em>与海螺<em>集团</em>签署战略合作协议\n2023-10-26 14:57\n来源：\n安徽省生态环境<em>集团</em>\n关键词：\n生态环境<em>集团</em>\n海螺<em>集团</em>\n环保产业\n收藏\n点赞\n分享\n订阅\n投稿\n我要投稿\n10月24日，安徽省\n生态环境<em>集团</em>",
            "publish_time": "2023-10-26T14:57:00+00:00",
            "source": "web_finance_news_db_20251225"
        },
        {
            "doc_id": 99371,
            "title": "中国山东重工集团与哈萨克斯坦阿鲁尔集团深化战略合作",
            "snippet": "中国山东重工<em>集团</em>与哈萨克斯坦阿鲁尔<em>集团</em>深化战略合作\n中国山东重工<em>集团</em>与哈萨克斯坦阿鲁尔<em>集团</em>深化战略合作 | 每经网\n每经网首页\n丨\n宏观\n丨\n金融\n丨\n公司\n丨\n视频\n丨\n券商\n丨\nIPO\n丨\n基金\n丨\n汽车\n丨\n房产\n丨\n新文化\n丨\n未来商业\n丨\n文创通\n丨\n城市\n丨\n每经商学院\n首发快讯\n每经网首页\n>\n首发快讯\n>\n        正文\n中国山东重工<em>集团</em>与哈萨克斯坦阿鲁尔<em>集团</em>深化战略合作",
            "highlight": "中国山东重工<em>集团</em>与哈萨克斯坦阿鲁尔<em>集团</em>深化战略合作\n中国山东重工<em>集团</em>与哈萨克斯坦阿鲁尔<em>集团</em>深化战略合作 | 每经网\n每经网首页\n丨\n宏观\n丨\n金融\n丨\n公司\n丨\n视频\n丨\n券商\n丨\nIPO\n丨\n基金\n丨\n汽车\n丨\n房产\n丨\n新文化\n丨\n未来商业\n丨\n文创通\n丨\n城市\n丨\n每经商学院\n首发快讯\n每经网首页\n>\n首发快讯\n>\n        正文\n中国山东重工<em>集团</em>与哈萨克斯坦阿鲁尔<em>集团</em>深化战略合作",
            "publish_time": "2023-09-03T15:03:56+00:00",
            "source": "web_finance_news_db_20260105"
        },
        {
            "doc_id": 90075,
            "title": "致远互联签约国资企业草滩集团 构建覆盖集团一体化协同平台",
            "snippet": "致远互联签约国资企业草滩<em>集团</em> 构建覆盖<em>集团</em>一体化协同平台\n致远互联签约甘肃建材<em>集团</em> 构建<em>集团</em>及11家下属单位一体化协同平台\n热文精选\n增速引领经济大市，成都2026年如何继续“挑大梁”？",
            "highlight": "致远互联签约国资企业草滩<em>集团</em> 构建覆盖<em>集团</em>一体化协同平台\n致远互联签约甘肃建材<em>集团</em> 构建<em>集团</em>及11家下属单位一体化协同平台\n热文精选\n增速引领经济大市，成都2026年如何继续“挑大梁”？",
            "publish_time": "2023-10-24T08:42:40+00:00",
            "source": "web_finance_news_db_20260102"
        },
        {
            "doc_id": 79994,
            "title": "招商建筑科技接连中标康泰集团、金辉集团智能化工程项目",
            "snippet": "招商建筑科技接连中标康泰<em>集团</em>、金辉<em>集团</em>智能化工程项目\n观点网讯：\n招商积余10月31日披露，旗下平台招商建筑科技近日接连中标康泰<em>集团</em>大厦自用办公智能化与机房系统工程、金辉<em>集团</em>“2023年-2025年福建公司智能化工程区域集采工程”。\n康泰<em>集团</em>大厦位于深圳南山区粤海街道科发路与科智东路交汇处，隶属康泰生物总部大楼建设项目。",
            "highlight": "招商建筑科技接连中标康泰<em>集团</em>、金辉<em>集团</em>智能化工程项目\n观点网讯：\n招商积余10月31日披露，旗下平台招商建筑科技近日接连中标康泰<em>集团</em>大厦自用办公智能化与机房系统工程、金辉<em>集团</em>“2023年-2025年福建公司智能化工程区域集采工程”。\n康泰<em>集团</em>大厦位于深圳南山区粤海街道科发路与科智东路交汇处，隶属康泰生物总部大楼建设项目。",
            "publish_time": "2023-10-31T18:59:00+00:00",
            "source": "web_finance_news_db_20260105"
        },
        {
            "doc_id": 89810,
            "title": "华蓝集团： 计提减值准备",
            "snippet": "华蓝<em>集团</em>： 计提减值准备\n2022年1至12月份，华蓝<em>集团</em>的营业收入构成为：服务业占比99.26%。\n华蓝<em>集团</em>的董事长是雷翔，男，66岁，学历背景为博士；总经理是莫海量，男，45岁，学历背景为硕士。\n截至发稿，华蓝<em>集团</em>市值为17亿元。\n道达号（daoda1997）“个股趋势”提醒：\n1.",
            "highlight": "华蓝<em>集团</em>： 计提减值准备\n2022年1至12月份，华蓝<em>集团</em>的营业收入构成为：服务业占比99.26%。\n华蓝<em>集团</em>的董事长是雷翔，男，66岁，学历背景为博士；总经理是莫海量，男，45岁，学历背景为硕士。\n截至发稿，华蓝<em>集团</em>市值为17亿元。\n道达号（daoda1997）“个股趋势”提醒：\n1.",
            "publish_time": "2023-10-24T16:32:45+00:00",
            "source": "web_finance_news_db_20251222"
        },
        {
            "doc_id": 85889,
            "title": "迅销集团、太平鸟集团、申洲国际等出席2023全球纺织碳中和国际峰会",
            "snippet": "迅销<em>集团</em>、太平鸟<em>集团</em>、申洲国际等出席2023全球纺织碳中和国际峰会\n迅销<em>集团</em>、太平鸟<em>集团</em>、申洲国际等出席2023全球纺织碳中和国际峰会-北极星环保网\n新闻\n政策\n招投标\n项目\n技术\n数据\n报告\n社区\n下载\n市场\n名企\n独家\n人物\n评论\n国际\n大气\n水处理\n固废\n垃圾发电\n环境修复\n环境监测\nVOCs\n节能\n环卫\n碳管家\n招聘\n学社\n直播\n设备网\n搜索历史\n清空\n水处理",
            "highlight": "迅销<em>集团</em>、太平鸟<em>集团</em>、申洲国际等出席2023全球纺织碳中和国际峰会\n迅销<em>集团</em>、太平鸟<em>集团</em>、申洲国际等出席2023全球纺织碳中和国际峰会-北极星环保网\n新闻\n政策\n招投标\n项目\n技术\n数据\n报告\n社区\n下载\n市场\n名企\n独家\n人物\n评论\n国际\n大气\n水处理\n固废\n垃圾发电\n环境修复\n环境监测\nVOCs\n节能\n环卫\n碳管家\n招聘\n学社\n直播\n设备网\n搜索历史\n清空\n水处理",
            "publish_time": "2023-10-16T17:21:00+00:00",
            "source": "web_finance_news_db_20251231"
        },
        {
            "doc_id": 84994,
            "title": "宝马集团宣布两项人事任命，高翔将出任宝马集团大中华区总裁",
            "snippet": "宝马<em>集团</em>宣布两项人事任命，高翔将出任宝马<em>集团</em>大中华区总裁\n自2018年以来，高乐开始负责宝马<em>集团</em>的中国业务，在其升任宝马<em>集团</em>董事后，高翔将全面负责宝马<em>集团</em>大中华区业务，包括协调宝马<em>集团</em>在中国的合资企业。",
            "highlight": "宝马<em>集团</em>宣布两项人事任命，高翔将出任宝马<em>集团</em>大中华区总裁\n自2018年以来，高乐开始负责宝马<em>集团</em>的中国业务，在其升任宝马<em>集团</em>董事后，高翔将全面负责宝马<em>集团</em>大中华区业务，包括协调宝马<em>集团</em>在中国的合资企业。",
            "publish_time": "2023-10-18T21:10:45+00:00",
            "source": "web_finance_news_db_20251228"
        },
        {
            "doc_id": 87455,
            "title": "",
            "snippet": "立\n10\n小米最新通报\n欢迎关注每日经济新闻APP\n0\n0\n相关信息\n关于我们\n版权声明\n联系我们\n广告投放\n友情链接\n关注我们\n辟谣专区\n加入我们\n每日经济新闻\n成都市天府科幻企业管理\n拟聘人员公示\nCopyright © 2026 每日经济新闻报社版权所有，未经许可不得转载使用，违者必究。\n广告热线  北京: 010-57613265， 上海: 021-61283008， 广州: 020-84201861， 深圳: 0755-83520159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 51",
            "highlight": "立\n10\n小米最新通报\n欢迎关注每日经济新闻APP\n0\n0\n相关信息\n关于我们\n版权声明\n联系我们\n广告投放\n友情链接\n关注我们\n辟谣专区\n加入我们\n每日经济新闻\n成都市天府科幻企业管理\n拟聘人员公示\nCopyright © 2026 每日经济新闻报社版权所有，未经许可不得转载使用，违者必究。\n广告热线  北京: 010-57613265， 上海: 021-61283008， 广州: 020-84201861， 深圳: 0755-83520159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 51",
            "publish_time": "",
            "source": ""
        },
        {
            "doc_id": 90979,
            "title": "贵州茅台回复“年内股价最大单日下跌”；京东与美的、小米等发起双十一“真低价”倡议；被曝所售海苔中有蟑螂，山姆回应丨大公司动态",
            "snippet": "贵州茅台回复“年内股价最大单日下跌”；京东与美的、<em>小米</em>等发起双十一“真低价”倡议；被曝所售海苔中有蟑螂，山姆回应丨大公司动态\n东风<em>集团</em>收购神龙公司三工厂\n10月19日，东风汽车<em>集团</em>股份有限公司披露交易公告称，东风<em>集团</em>购买神龙汽车有限公司位于中国武汉和襄阳的特定土地使用权、建筑物和构筑物。根据协议条款，公司同意以人民币17.14亿元对价收购目标资产，神龙公司同意以人民币17.14亿元出售目标资产。\n代工<em>小米</em>汽车？",
            "highlight": "贵州茅台回复“年内股价最大单日下跌”；京东与美的、<em>小米</em>等发起双十一“真低价”倡议；被曝所售海苔中有蟑螂，山姆回应丨大公司动态\n东风<em>集团</em>收购神龙公司三工厂\n10月19日，东风汽车<em>集团</em>股份有限公司披露交易公告称，东风<em>集团</em>购买神龙汽车有限公司位于中国武汉和襄阳的特定土地使用权、建筑物和构筑物。根据协议条款，公司同意以人民币17.14亿元对价收购目标资产，神龙公司同意以人民币17.14亿元出售目标资产。\n代工<em>小米</em>汽车？",
            "publish_time": "2023-10-19T19:15:04+00:00",
            "source": "web_finance_news_db_20251225"
        },
        {
            "doc_id": 94024,
            "title": "",
            "snippet": "10\n小米最新通报\n欢迎关注每日经济新闻APP\n0\n0\n相关信息\n关于我们\n版权声明\n联系我们\n广告投放\n友情链接\n关注我们\n辟谣专区\n加入我们\n每日经济新闻\n成都市天府科幻企业管理\n拟聘人员公示\nCopyright © 2026 每日经济新闻报社版权所有，未经许可不得转载使用，违者必究。\n广告热线  北京: 010-57613265， 上海: 021-61283008， 广州: 020-84201861， 深圳: 0755-83520159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 5101",
            "highlight": "10\n小米最新通报\n欢迎关注每日经济新闻APP\n0\n0\n相关信息\n关于我们\n版权声明\n联系我们\n广告投放\n友情链接\n关注我们\n辟谣专区\n加入我们\n每日经济新闻\n成都市天府科幻企业管理\n拟聘人员公示\nCopyright © 2026 每日经济新闻报社版权所有，未经许可不得转载使用，违者必究。\n广告热线  北京: 010-57613265， 上海: 021-61283008， 广州: 020-84201861， 深圳: 0755-83520159， 成都: 028-86512112\n互联网新闻信息服务许可证：51120190017\n网站备案号：蜀ICP备19004508号-3\n川公网安备 5101",
            "publish_time": "",
            "source": ""
        },
        {
            "doc_id": 85180,
            "title": "华阳集团：接受东吴证券等机构调研",
            "snippet": "华阳<em>集团</em>：接受东吴证券等机构调研\n上一篇文章\n三棵树：截至本公告披露日，公司及子公司担保余额为人民币约31.12亿元\n返回每经网首页\n下一篇文章\n瑞联新材：公司EUV光刻胶单体产品中试批次已通过客户验证 且有百公斤级销售\n相关文章\n华阳<em>集团</em>：接受中信证券等机构调研\n华阳<em>集团</em>：接受开源证券等机构调研\n华阳国际：接受天风证券等机构调研\n华阳<em>集团</em>",
            "highlight": "华阳<em>集团</em>：接受东吴证券等机构调研\n上一篇文章\n三棵树：截至本公告披露日，公司及子公司担保余额为人民币约31.12亿元\n返回每经网首页\n下一篇文章\n瑞联新材：公司EUV光刻胶单体产品中试批次已通过客户验证 且有百公斤级销售\n相关文章\n华阳<em>集团</em>：接受中信证券等机构调研\n华阳<em>集团</em>：接受开源证券等机构调研\n华阳国际：接受天风证券等机构调研\n华阳<em>集团</em>",
            "publish_time": "2023-10-31T18:06:21+00:00",
            "source": "web_finance_news_db_20260109"
        }
    ],
    "msg": "success"
}
```



```
curl -X DELETE "http://localhost:9200/news_chunk_index"

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
