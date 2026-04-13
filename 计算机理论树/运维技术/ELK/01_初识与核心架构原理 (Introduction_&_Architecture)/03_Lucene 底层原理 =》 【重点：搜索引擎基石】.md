
Apache Lucene 是一款高性能、可扩展的开源全文搜索引擎库，是 Elasticsearch 的底层核心，Elasticsearch 的所有搜索、聚合、排序能力均基于 Lucene 的核心机制封装实现，是整个 Elastic Stack 的搜索引擎基石。其核心设计围绕**倒排索引体系**与**正排索引优化**两大核心方向展开。

### 1.3.1 倒排索引（Inverted Index）

倒排索引是全文检索的核心数据结构，其核心逻辑是建立**词条（Term）→ 文档**的映射关系，与正排索引的**文档→词条**映射形成互补。正排索引适用于通过文档 ID 获取文档内容的场景，而倒排索引专门解决通过词条快速定位包含该词条的所有文档的全文检索需求。

Lucene 的倒排索引采用三级存储结构，从底层到上层依次为 **Posting List → Term Dictionary → Term Index** ，三层结构协同实现了低内存占用、高查询效率的全文检索能力。

#### 1.3.1.1 Posting List（倒排表）

Posting List 是倒排索引的最底层存储结构，存储了包含某个特定 Term 的所有文档 ID，以及该 Term 在对应文档中的词频（TF）、偏移量、位置等元数据。

- 核心作用：通过 Term 直接获取匹配的全量文档 ID 集合，为后续检索结果筛选、相关性打分、短语匹配、高亮展示提供数据支撑；
- 性能优化：Posting List 采用增量编码、FOR（Frame Of Reference）、Roaring Bitmap 等压缩算法，大幅降低磁盘存储占用与内存加载开销，同时提升文档 ID 集合的交并集计算效率。

#### 1.3.1.2 Term Dictionary（词条字典）

Term Dictionary 存储了文档集中所有不重复的 Term，并建立了 Term 与对应 Posting List 存储位置的映射关系。

- 核心设计：Term Dictionary 按照字典序排序，支持二分查找，可快速定位目标 Term；
- 架构局限：当文档量级达到千万级以上时，Term 的数量可达到亿级，全量加载 Term Dictionary 至内存会产生极大的内存开销，因此引入 Term Index 作为二级索引，平衡内存占用与查询效率。

#### 1.3.1.3 Term Index（词条索引）

Term Index 是 Term Dictionary 的二级索引，采用 FST（Finite State Transducer，有限状态转换器）数据结构实现，核心特性是对前缀相同的 Term 进行合并存储，实现了极低的内存占用与极高的查找效率。

- 核心作用：Term Index 不存储全量 Term，仅存储 Term 的前缀与对应 Term Dictionary 的磁盘偏移量的映射关系，通过 Term Index 可快速定位目标 Term 在 Term Dictionary 中的存储位置，避免全量扫描 Term Dictionary，将 Term 查找的时间复杂度降至 O (log n) 级别；
- 核心优势：FST 结构的 Term Index 可全量加载至内存，无需磁盘 IO 即可完成 Term 的快速定位，彻底解决了大词条量下的检索性能瓶颈。

#### 倒排索引完整检索流程

1. 用户输入查询文本，经分词器处理为多个独立 Term；
2. 针对每个 Term，通过内存中的 Term Index 定位其在 Term Dictionary 中的磁盘偏移量；
3. 读取 Term Dictionary 中对应 Term 的元数据，获取其关联的 Posting List 的存储地址；
4. 加载 Posting List，获取包含该 Term 的全量文档 ID 集合；
5. 对多个 Term 对应的文档 ID 集合执行交并集计算，得到最终匹配的文档 ID 列表，完成检索的第一阶段。

### 1.3.2 正排索引与 Doc Values

倒排索引完美适配全文检索的 “Term 找文档” 场景，但对于排序、聚合分析这类 “文档找字段值” 的场景，倒排索引性能极低 —— 需遍历 Posting List 并逐文档随机读取字段值，产生大量磁盘 IO 开销。为解决该问题，Lucene 引入了正排索引体系，其核心实现为**Doc Values**。

Doc Values 是 Elasticsearch 基于 Lucene 实现的**列式存储正排索引**，在文档写入时与倒排索引同步生成、持久化至磁盘，默认对所有非 text 类型字段自动开启。

其核心设计与特性如下：

1. 列式存储架构：按字段组织数据，而非按文档组织数据。针对排序、聚合场景，仅需加载目标字段的全量数据，无需读取文档的其他字段，大幅降低 IO 开销，提升批量计算效率；
2. 预计算与持久化：文档写入时同步生成 Doc Values，避免查询时的实时计算开销，将计算压力从查询阶段转移至写入阶段，适配海量数据下的低延迟聚合查询需求；
3. 压缩优化：采用增量编码、字典编码、LZ4 压缩等算法，降低磁盘存储占用与内存加载开销；
4. 与倒排索引的协同：全文检索阶段通过倒排索引获取匹配的文档集，排序、聚合阶段通过 Doc Values 对文档集进行高效的批量计算，二者共同构成 Elasticsearch 的核心数据处理底座。
