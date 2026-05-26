# 向量搜索实现梳理（代码导向）

本文面向开发者，按“入口 -> 映射/索引 -> 查询执行 -> 优化策略 -> 测试样例”梳理 Elasticsearch 中向量搜索的关键实现位置，帮助快速定位代码。

## 1. 查询入口（DSL 与请求体）

### `knn` query（Query DSL）
- 入口类：`KnnVectorQueryBuilder`（`name = "knn"`）。
- 负责解析 `field`、`query_vector`、`k`、`num_candidates`、`visit_percentage`、`filter`、`query_vector_builder`、`rescore_vector` 等参数。
- 支持自动 prefiltering（并对嵌套查询有显式限制说明）。

代码位置：
- `server/src/main/java/org/elasticsearch/search/vectors/KnnVectorQueryBuilder.java`

### 搜索请求级 `knn` 块
- 入口类：`KnnSearchBuilder`，用于 search request 中的 kNN 段。
- 和 `KnnVectorQueryBuilder` 类似，负责参数解析、序列化与 rewrite。

代码位置：
- `server/src/main/java/org/elasticsearch/search/vectors/KnnSearchBuilder.java`

## 2. 向量字段建模与索引路径

### `dense_vector` 字段核心
- 核心 mapper：`DenseVectorFieldMapper`。
- 负责 dense vector 的 mapping 参数、向量编码、索引字段构造，以及不同向量格式/策略的接入。
- 同时定义了过滤场景下 HNSW 搜索启发式（`FANOUT` / `ACORN`）及相关 index setting。

代码位置：
- `server/src/main/java/org/elasticsearch/index/mapper/vectors/DenseVectorFieldMapper.java`

### 向量 codec / format 层（按版本与能力拆分）
- 目录：`server/src/main/java/org/elasticsearch/index/codec/vectors/`
- 含多个子实现（例如 `es93`、`es94`、`diskbbq`），用于承接不同向量存储与检索格式。

## 3. 查询执行主干（Lucene kNN 的 ES 包装）

目录：`server/src/main/java/org/elasticsearch/search/vectors/`

常见职责分层：
- Query 包装层：`ESKnnFloatVectorQuery`、`ESKnnByteVectorQuery`、`DenseVectorQuery` 等。
- 召回/候选控制：`RescoreKnnVectorQuery`、`KnnScoreDocQuery`。
- Collectors：`MaxScoreTopKnnCollector`、`BulkKnnCollector`、`AdaptiveHnswQueueSaturationCollector`。
- 过滤和父子文档相关：`Diversifying*` 系列 query/collector。

## 4. IVF 与访问策略

### IVF 搜索策略
- `IVFKnnSearchStrategy` 继承 Lucene `KnnSearchStrategy`。
- 关键参数：`visitRatio`、`numCands`、`k`。
- 通过 `nextVectorsBlock()` 在分块处理中更新竞争阈值，协调 collector 与全局阈值累积器，减少低价值比较。

代码位置：
- `server/src/main/java/org/elasticsearch/search/vectors/IVFKnnSearchStrategy.java`

## 5. Query Vector 构建（异步）

- `QueryVectorBuilderAsyncAction` 负责通过 `QueryVectorBuilder` 异步构建查询向量。
- 在 rewrite / 执行前把外部构建器产出的向量转成实际 `float[]`，并对 `null` 返回做显式校验。

代码位置：
- `server/src/main/java/org/elasticsearch/search/vectors/QueryVectorBuilderAsyncAction.java`

## 6. 性能相关实现点（建议重点阅读）

- SIMD 向量计算库：`libs/simdvec/`（包含多种 scorer / util）。
- Dense vector 的编码与 doc values 读取：`index/mapper/vectors/` 下相关类。
- quantization / disk-bbq 相关格式与搜索测试：`index/codec/vectors/` 与 `x-pack/plugin/.../search.vectors/*.yml`。

## 7. 典型测试入口

可从以下目录理解行为覆盖面：
- `x-pack/plugin/src/yamlRestTest/resources/rest-api-spec/test/search.vectors/`
- `modules/lang-painless/src/yamlRestTest/resources/rest-api-spec/test/painless/`（dense_vector 脚本访问与函数）

---

## 建议阅读顺序

1. `DenseVectorFieldMapper`
2. `KnnVectorQueryBuilder` / `KnnSearchBuilder`
3. `ESKnn*Query` + `RescoreKnnVectorQuery`
4. `IVFKnnSearchStrategy` + collectors
5. `libs/simdvec` 中 scorer 与 util

这样可以先建立“参数语义 -> 索引能力 -> 执行路径 -> 性能优化”的完整链路。
