# Nebula Graph 索引详解


> `index not found`?找不到索引？为什么我要创建 Nebula Graph 索引？什么时候要用到 Nebula Graph 原生索引，一文把这些搞清楚。

Nebula Graph 的索引其实和传统的关系型数据库中的索引很像，但是又有一些容易让人疑惑的区别。刚开始了解 Nebula 的同学会疑惑于：

- 不清楚 Nebula Graph 图数据库中的索引到的是什么概念
- 我应该什么时候使用 Nebula Graph 索引
- Nebula Graph 索引怎么影响到写入性能

这篇文章里，我们就把这些问题回答好。

## 到底 Nebula Graph 索引是什么

简而言之，Nebula Graph 索引是用来，且只用来针对纯属性条件出发查询场景的

- 图游走（walk）查询中的属性条件过滤不需要它
- 纯属性条件出发查询（注：非采样情况）必须创建索引

### 纯属性条件出发查询

我们知道在传统关系型数据库中，索引是对表数据的一个针对特定**列**重排序的副本，它用来**加速特定列过滤条件的读查询**并带来了额外的数据写入（加速而非这样查询的必须前提）。

在 Nebula Graph 图数据库里，索引则是对**点、边特定属性数据**重排序的副本，用来提供**纯属性条件出发查询**（如下边的查询：从只给定了点边属性条件，而非点的 ID 出发去获取图数据）

```sql
#### 必须 Nebula Graph 索引存在的查询

# query 0 纯属性条件出发查询
LOOKUP ON tag1 WHERE col1 > 1 AND col2 == "foo" \
    YIELD tag1.col1 as col1, tag1.col3 as col3;

# query 1 纯属性条件出发查询
MATCH (v:player { name: 'Tim Duncan' })-->(v2:player) \
        RETURN v2.player.name AS Name;
```

上边这两个纯属性条件出发查询就是字面意思的”根据给定的属性条件获取点或者边本身“ ，反面的例子则是给定了点的 ID：

```sql
#### 不基于索引的查询

# query 2, 从给定的点做的游走查询 vertex VID: "player100"

GO FROM "player100" OVER follow REVERSELY \
        YIELD src(edge) AS id | \
    GO FROM $-.id OVER serve \
        WHERE properties($^).age > 20 \
        YIELD properties($^).name AS FriendOf, properties($$).name AS Team;

# query 3, 从给定的点做的游走查询 vertex VID: "player101" 或者 "player102"

MATCH (v:player { name: 'Tim Duncan' })--(v2) \
        WHERE id(v2) IN ["player101", "player102"] \
        RETURN v2.player.name AS Name;
```



我们仔细看前边的 `query 1` 和 `query 3`，尽管语句中条件都有针对 tag 为 player 的过滤： `{ name: 'Tim Duncan' }` ：

- `query 3`之中不需要索引，因为它可以：
  - 更直接的从已知的 `v2` 顶点： `["player101", "player102"]` 
  - 向外扩展、游走（GetNeighbors() 获得边的另一端的点，然后GetVertices() 得到下一跳的 `v`），根据 v.player.name 过滤掉不要的数据
  
- `query 1` 则不同，它因为没有任何给定的顶点 ID：
  - 只能从属性条件入手，`{ name: 'Tim Duncan' }`，在按照 name 排序了的索引数据中先找到符合的点：IndexScan() 得到 `v`
  - 然后再从 `v` 做 GetNeighbors() 获得边的另一端 的 `v2` ，在通过 GetVertices() 去获得下一跳 `v2` 中的数据

其实，这里的关键就是在于是查询是否存在给定的顶点 ID（Vertex ID），下边两个查询的执行计划里更清晰地比较了他们的区别：

| `query 1`, 需要基于索引，纯属性条件出发查询          | `query 3`, 从已知 VID，不需要索引                          |
| ---------------------------------------------------- | ---------------------------------------------------------- |
| ![query-based-on-index](./query-based-on-index.webp) | ![query-requires-no-index](./query-requires-no-index.webp) |



### 为什么纯属性条件出发查询里必须要索引呢？

因为 Nebula Graph 在存储数据的时候，它的结构是面向分布式与关联关系设计的，类似表结构数据库中无索引的全扫描条件搜索实际上更加昂贵，所以设计上被有意禁止了。



> 注: 如果不追求全部数据，只要采样一部分，3.0 里之后是支持不强制索引 LIMIT <n> 的情况的，如下查询（有 LIMIT）不需要索引：
>
> ```cypher
> MATCH (v:player { name: 'Tim Duncan' })-->(v2:player) \
>         RETURN v2.player.name AS Name LIMIT 3;
> ```

### 为什么只有纯属性条件出发查询

我们比较一下正常的图查询 **graph-queries** 和纯属性条件出发查询 **pure-prop-condition queries**：

- graph-queries： 如 `query 2` 、 `query 3`是沿着边一路找到特定路径条件的扩展游走
- pure-prop-condition queries：如 `query 0` and `query 1`  是只通过一定属性条件（或者是无限制条件）找到满足的点、边

而在 Nebula Graph 里，graph-queries 在扩展的时候，图的原始数据已经按照 VID（点和边都是）排序过了（或者说在数据里已经索引过了），这个排序带来连续存储（物理上临接）使得扩展游走本身就是优化、很快的。

所以：

> Nebula Graph 索引是为了从给定属性条件查点、边的一份属性数据的排序。

## 一些 Nebula Graph 索引的设计细节

为了更好理解索引的限制、代价、能力，咱们来解释更多他的细节

- Nebula Graph 索引是在本地（不是分开、中心化）和点数据被一起存储、分片的。
- 它只支持左匹配
  - 因为底层是 RocksDB Prefix Scan

- 性能代价:

  - 写入时候的路径：不只是多一分数据，为了保证一致性，还有昂贵的读操作
  - 读路径：基于规则的优化选择索引，Fan Out 到所有 StorageD

这些信息也在我的[手绘图和视频](https://www.siwei.io/sketch-notes/)里可以看到：

![](https://www.siwei.io/sketches/nebula-index-demystified/nebula-index-demystified.webp)

> 因为左匹配的设计，在有更复杂的针对纯属性条件出发查询里涉及到通配、REGEXP这样的全文搜索情况，Nebula Graph 提供了全文索引的功能，它是利用 Raft Listener 去异步将数据写到外部 Elasticsearch 集群之中，并在查询的时候去查 ES 去做到的，见[文档](https://docs.nebula-graph.com.cn/3.0.0/4.deployment-and-installation/6.deploy-text-based-index/2.deploy-es/)。

在这个手绘图中，我们还可以看出

- Write path
  - 写入索引数据是同步操作的
- Read path
  - 这部分画了一个 RBO 的例子，查询里的规则假设 col2 相等匹配排在左边的情况下，性能优于 col1 的大小比较匹配，所以选择了第二个索引
  - 选好了索引之后，扫描索引的请求被 fan out 到存储节点上，这其中有些过滤条件比如 top n 是可以下推的


结论：

- 因为写入的代价，只有必须用索引的时候采用，如果采样查询能满足读的要求，可以不创建索引而用 LIMIT <n>。
- 索引有左匹配的限制
  - 符合查询的顺序要仔细设计
  - 有时候需要使用全文索引 [full-text index](https://docs.nebula-graph.com.cn/3.0.0/4.deployment-and-installation/6.deploy-text-based-index/2.deploy-es/)。




## 索引的使用

具体要参考[索引文档](https://docs.nebula-graph.io/3.0.0/3.ngql-guide/14.native-index-statements/)一些要点是：

- 在 tag 或者 edge type 上针对想要被条件反查点边的属性创建索引

  - `CREATE INDEX`

- 创建索引之后的索引部分数据会同步写入，但是如果创建索引之前已经有的点边数据对应的索引是需要明确指定去创建的，这是一个异步的 job：

  - `REBUILD INDEX`

- 触发了异步的 `REBUILD INDEX` 之后，可以查询状态：

  - `SHOW INDEX STATUS`

- 利用到索引的查询可以是 LOOKUP，并且常常可以借助管道符在此之上做拓展查询（Graph Query）：

  ```sql
  LOOKUP ON player \
    WHERE player.name == "Kobe Bryant"\
    YIELD id(vertex) AS VertexID, properties(vertex).name AS name |\
    GO FROM $-.VertexID OVER serve \
    YIELD $-.name, properties(edge).start_year, properties(edge).end_year, properties($$).name;
  ```

- 也可以是 MATCH ，这里边 `v` 是通过索引得到的，而 `v2` 则是在数据（非索引）部分拓展查询获得的。

  ```cypher
  MATCH (v:player{name:"Tim Duncan"})--(v2:player) \
    RETURN v2.player.name AS Name;
  ```



## 回顾

- Nebula Graph 索引在只提供属性条件情况下通过对属性的排序副本扫描查点、边
- Nebula Graph 索引**不是**用来图拓展查询的
- Nebula Graph 索引是左匹配，不是用来做模糊全文搜索的
- Nebula Graph 索引在写入时候有性能代价
- 记得如果创建 Nebula Graph 索引之前已经有相应点边上的数据，要重建索引

Happy Graphing!

Feture image credit to [Alina](https://unsplash.com/photos/ZiQkhI7417A)

