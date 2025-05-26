---
draft: false
date:
  created: 2025-01-03
  updated: 2025-01-12
  updated: 2025-01-13
  updated: 2025-05-26
categories:
  - Learning
tags:
  - LLM
  - Vector Storage
authors:
  - zhazi
---

# LLM 学习：Step2 - Milvus


**之前部署的 TinyRAG 部署使用 JSON 文件存储向量化结果。这种方案虽然易于实现，但可以预见的是，随着知识库规模的扩大，性能瓶颈将逐渐显现：查询效率下降、存储空间占用过大、数据维护困难等问题接踵而至。而这，就是各种向量数据库发挥作用的地方了。**

Milvus 是一个高性能、高扩展性的**向量数据库**，Milvus 支持多种数据类型和搜索功能，包括 ANN 搜索、过滤搜索和全文搜索等。Milvus 的设计考虑了硬件优化和高效的搜索引擎，因而具有良好的性能。此外，Milvus 还提供了多种部署模式，以适应不同的数据规模和应用需求。

??? info "参考文献"

    - [【【上集】向量数据库技术鉴赏】](https://www.bilibili.com/video/BV11a4y1c7SW)
    - [【【下集】向量数据库技术鉴赏】](https://www.bilibili.com/video/BV1BM4y177Dk)

## 为什么需要向量数据库？
这个问题可以分为两部分：为什么需要向量存储？以及为什么不使用传统数据库做向量存储？

**为什么需要向量存储？**原因很简单，深度学习领域中常常涉及到对数据的检索，而这些数据通常被编码为特征向量，因此需要有一种方式能把向量存储下来。

**为什么不使用传统数据库？**因为传统的关系型数据库中数据是在一个集合中的，用户通过各种条件设置对集合筛选出数据。而向量检索中的数据是在一个空间中的，用户需要的是找到距离输入向量最近的那条数据。传统数据库提供的服务和向量检索场景中用户的需求不匹配。

## 向量数据库原理

参考文献讲的很好，暂时没必要在这里画蛇添足。这里居然看到了之前研究过的 HNSW，图论还是有点用的嘛。

??? notes "ANN"

    向量数据库主要提供的服务是**查询与输入向量最接近的结果**。然而，要找到最正确的结果需要遍历数据库中的所有向量，这显然是不合适的。在精度和开销的权衡中，就有了近似最近邻（ANN）的概念。我们不再要求找到最接近的结果，只要找到一个大致接近的结果就好。


## 升级 TinyRAG 的存储方案

这里需要改动两个逻辑：1. 处理文档库然后存到向量数据库中（创建数据库）的逻辑；2. 访问数据库查找相关的文本片段（使用数据库）的逻辑。

### 创建数据库

在 `config.yml` 中添加 Milvus 相关的配置：

```yaml title="config.yml"
milvus:
  db_name: local_knowledge
  collection:
    name: myblog
    docs: ../../myblog/docs/blog/
  search_limit: 2
```

新建 `create_vector.py` 文件，存放创建数据库的逻辑：

```python title="create_vector.py" linenums="1"
from omegaconf import OmegaConf
from pymilvus import MilvusClient

from TinyRAG.utils import ReadFiles
from TinyRAG.Embeddings import ZhipuEmbedding


# NOTE: 读取配置文件
cfg = OmegaConf.load('./config.yml')

# NOTE: 创建数据库 (如果存在则重建)
client = MilvusClient(cfg.milvus.db_name + ".db")
if client.has_collection(collection_name=cfg.milvus.collection.name):
    client.drop_collection(collection_name=cfg.milvus.collection.name)

client.create_collection(
    collection_name=cfg.milvus.collection.name,
    dimension=cfg.vec_dimension,
)

# NOTE: 读取文档库，使用智谱的 API 把文档片段编码为向量
docs = ReadFiles(cfg.milvus.collection.docs).get_content(max_token_len=600, cover_content=150) 
embedding_model = ZhipuEmbedding(dimensions=cfg.vec_dimension)
vectors = embedding_model(docs)

# NOTE: 把数据插入数据库
data = [
    {"id": i, "vector": vectors[i], "text": docs[i], "subject": 'blog'}
    for i in range(len(vectors))
]
res = client.insert(collection_name=cfg.milvus.collection.name, data=data)
print(f"插入了 {res['insert_count']} 篇文档，ID 为: {res['ids']}。操作耗时: {res['cost']} 毫秒")
```

### 使用数据库

在 `main.py` 中增加连接数据库和检索数据库的逻辑：

```python title="main.py" linenums="12"
# NOTE: 连接数据库
client = MilvusClient(cfg.milvus.db_name + ".db")
if client.has_collection(collection_name=cfg.milvus.collection.name):
    client.create_collection(
        collection_name=cfg.milvus.collection.name,
        dimension=cfg.vec_dimension,
    )
else:
    print(f"警告: 集合 {cfg.milvus.collection.name} 不存在")

embedding_model = ZhipuEmbedding(dimensions = cfg.vec_dimension) # 创建EmbeddingModel

def handle_client(conn):
    try:
        question = conn.recv(1024).decode('utf-8').strip()
        if not question:
            return

        # NOTE: 检索数据库
        result = client.search(
            collection_name=cfg.milvus.collection.name,
            data=embedding_model([question]),
            filter="subject == 'blog'",
            limit=cfg.milvus.search_limit,
            output_fields=["text", "subject"],
        )
        if len(result) == 0:
            content = "知识库中未查询到结果"
        else:
            content = (
                "知识库中查询到如下结果:\n"
                + "\n".join([f"{item['entity']['text']}" for item in result[0]])
                + "\n"
            )
        model = DeepSeekChat()
        response = model.chat(question, [], content) + '\n'
        conn.send(response.encode('utf-8'))
    finally:
        conn.close()
```
