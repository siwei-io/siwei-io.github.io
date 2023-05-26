# Nebulagraph Artificial Intelligence Suite


> 介绍新项目！ ng_ai：NebulaGraph 的图算法套件，好用的 NebulaGraph 的 high-level Python Algorithm API，它的目标是让 NebulaGraph 的数据科学家用户能够用很少的代码量执行图上的算法相关的任务。

<!--more-->


## Nebulagraph AI 套件

这周，NebulaGraph 3.5.0 [发布啦](https://docs.nebula-graph.com.cn/3.5.0/20.appendix/release-notes/nebula-comm-release-note/)，@**[whitewum](https://github.com/whitewum)** 吴老师建议我们把而之前一段时间 NebulaGraph 社区里开启的新项目 [ng_ai](https://github.com/wey-gu/nebulagraph-ai) 公开给大家，本文就是第一篇介绍 ng_ai 的文章！

### ng_ai 是什么

ng_ai 的全名是：Nebulagraph AI Suite，顾名思义，它是在 NebulaGraph 之上跑算法的 Python 套件，希望能给 NebulaGraph 的数据科学家用户一个自然、简洁的高级 API，用很少的代码量执行图上的算法相关的任务。

在 ng_ai 这个开源项目里，我们希望快速迭代、公开讨论、演进它，而这背后的目标是：

> Simplifying things in surprising ways.

### ng_ai 的特点

为了让 NebulaGraph 社区的同学拥有顺滑的算法体验，ng_ai 有以下特点：

- 与 NebulaGraph 紧密结合，方便从其中读、写图数据
- 支持多引擎、后端，目前支持 Spark（NebulaGraph Algorithm）、NetworkX，之后会支持 [DGL](https://www.dgl.ai/)、[PyG](https://pytorch-geometric.readthedocs.io/en/latest/)
- 友好、符合直觉的 API 设计
- 与 NebulaGraph 的 UDF 无缝结合，支持从 Query 中调用 ng_ai 任务
- 友好的自定义算法接口，方便用户自己实现算法（尚未完成）
- 一键试玩环境（基于 Docker Extention）

## 我可以用 ng_ai 干什么

### 跑分布式 pagerank 算法

如果在一个大图上，基于 Nebula-Algorithms 分布式地跑 pagerank 算法，我们可以这么做：

```python
from ng_ai import NebulaReader

# read data with spark engine, scan mode
reader = NebulaReader(engine="spark")
reader.scan(edge="follow", props="degree")
df = reader.read()

# run pagerank algorithm
pr_result = df.algo.pagerank(reset_prob=0.15, max_iter=10)
```

### 写回算法结果到 NebulaGraph

假设我们要跑一个 label propagation 算法，然后把结果写回 NebulaGraph，我们可以这么做：

先确保要写回 TAG 的 schema 已经创建好了，写到 label_propagation.cluster_id 字段里：

```sql
CREATE TAG IF NOT EXISTS label_propagation (
    cluster_id string NOT NULL
);
```

我们先执行算法：
   
```python
df_result = df.algo.label_propagation()

```

再看一下结果的 schema：

```python
df_result.printSchema()

root
 |-- _id: string (nullable = false)
 |-- lpa: string (nullable = false)
```

然后，代码里这么写，我们把 lpa 的结果写回 NebulaGraph 中的 cluster_id 字段里（`{"lpa": "cluster_id"}`）：

```python
from ng_ai import NebulaWriter
from ng_ai.config import NebulaGraphConfig

config = NebulaGraphConfig()
writer = NebulaWriter(
    data=df_result, sink="nebulagraph_vertex", config=config, engine="spark"
)

# map column louvain into property cluster_id
properties = {"lpa": "cluster_id"}

writer.set_options(
    tag="label_propagation",
    vid_field="_id",
    properties=properties,
    batch_size=256,
    write_mode="insert",
)
# write back to NebulaGraph
writer.write()

```

最后，我们可以验证一下结果啦：

```cypher
USE basketballplayer;
MATCH (v:label_propagation)
RETURN id(v), v.label_propagation.cluster_id LIMIT 3;
```

结果：

```SQL
+-------------+--------------------------------+
| id(v)       | v.label_propagation.cluster_id |
+-------------+--------------------------------+
| "player103" | "player101"                    |
| "player113" | "player129"                    |
| "player121" | "player129"                    |
+-------------+--------------------------------+
```

更详细的例子参考：[ng_ai/examples](https://github.com/wey-gu/nebulagraph-ai/blob/main/examples/spark_engine.ipynb)

#### 通过 nGQL 调用算法

从 NebulaGraph 3.5.0 之后，我们可以写自己的 UDF 来从 nGQL 里调用自己实现的函数，ng_ai 也用这个能力来实现了一个 ng_ai 函数，它可以从 nGQL 里调用 ng_ai 的算法，例如：

```sql
-- Prepare the write schema
USE basketballplayer;
CREATE TAG IF NOT EXISTS pagerank(pagerank string);
:sleep 20;
-- Call with ng_ai()
RETURN ng_ai("pagerank", ["follow"], ["degree"], "spark", {space: "basketballplayer", max_iter: 10}, {write_mode: "insert"})
```

更详细的例子参考：[ng_ai/examples](https://github.com/wey-gu/nebulagraph-ai/blob/main/examples/ng_ai_from_ngql_udf.ipynb)

#### 单机运行算法

在单机、本地的环境里，ng_ai 支持基于 NetworkX 运行算法，例如：

读取图为 ng_ai graph 对象：

```python
from ng_ai import NebulaReader
from ng_ai.config import NebulaGraphConfig

# read data with nebula/networkx engine, query mode
config_dict = {
    "graphd_hosts": "graphd:9669",
    "user": "root",
    "password": "nebula",
    "space": "basketballplayer",
}
config = NebulaGraphConfig(**config_dict)
reader = NebulaReader(engine="nebula", config=config)
reader.query(edges=["follow", "serve"], props=[["degree"], []])
g = reader.read()
```

查看、画图：

```python
g.show(10)
g.draw()
```

运行算法：

```python
pr_result = g.algo.pagerank(reset_prob=0.15, max_iter=10)
```

写回 NebulaGraph：

```python
from ng_ai import NebulaWriter

writer = NebulaWriter(
    data=pr_result,
    sink="nebulagraph_vertex",
    config=config,
    engine="nebula",
)

# properties to write
properties = ["pagerank"]

writer.set_options(
    tag="pagerank",
    properties=properties,
    batch_size=256,
    write_mode="insert",
)
# write back to NebulaGraph
writer.write()
```

其他算法：

```python
# get all algorithms
g.algo.get_all_algo()

# get help of each algo
help(g.algo.node2vec)

# call the algo
g.algo.node2vec()
```

更详细的例子参考：[ng_ai/examples](https://github.com/wey-gu/nebulagraph-ai/blob/main/examples/networkx_engine.ipynb)

#### 可视化图算法结果

再演示一个 NetworkX 引擎情况下，计算 Louvain、PageRank 并可视化的例子：

先执行两个算法：

```python
pr_result = g.algo.pagerank(reset_prob=0.15, max_iter=10)
louvain_result = g.algo.louvain()
```

这次我们手写一个好看一点的画图函数：

```python
from matplotlib.colors import ListedColormap


def draw_graph_louvain_pr(G, pr_result, louvain_result, colors=["#1984c5", "#22a7f0", "#63bff0", "#a7d5ed", "#e2e2e2", "#e1a692", "#de6e56", "#e14b31", "#c23728"]):
    # Define positions for the nodes
    pos = nx.spring_layout(G)

    # Create a figure and set the axis limits
    fig, ax = plt.subplots(figsize=(35, 15))
    ax.set_xlim(-1, 1)
    ax.set_ylim(-1, 1)

    # Create a colormap from the colors list
    cmap = ListedColormap(colors)

    # Draw the nodes and edges of the graph
    node_colors = [louvain_result[node] for node in G.nodes()]
    node_sizes = [70000 * pr_result[node] for node in G.nodes()]
    nx.draw_networkx_nodes(G, pos=pos, ax=ax, node_color=node_colors, node_size=node_sizes, cmap=cmap, vmin=0, vmax=max(louvain_result.values()))

    nx.draw_networkx_edges(G, pos=pos, ax=ax, edge_color='gray', width=1, connectionstyle='arc3, rad=0.2', arrowstyle='-|>', arrows=True)

    # Extract edge labels as a dictionary
    edge_labels = nx.get_edge_attributes(G, 'label')

    # Add edge labels to the graph
    for edge, label in edge_labels.items():
        ax.text((pos[edge[0]][0] + pos[edge[1]][0])/2,
                (pos[edge[0]][1] + pos[edge[1]][1])/2,
                label, fontsize=12, color='black', ha='center', va='center')

    # Add node labels to the graph
    node_labels = {n: G.nodes[n]['label'] if 'label' in G.nodes[n] else n for n in G.nodes()}
    nx.draw_networkx_labels(G, pos=pos, ax=ax, labels=node_labels, font_size=12, font_color='black')

    # Add colorbar for community colors
    sm = plt.cm.ScalarMappable(cmap=cmap, norm=plt.Normalize(vmin=0, vmax=max(louvain_result.values())))
    sm.set_array([])
    cbar = plt.colorbar(sm, ax=ax, ticks=range(max(louvain_result.values()) + 1), shrink=0.5)
    cbar.ax.set_yticklabels([f'Community {i}' for i in range(max(louvain_result.values()) + 1)])

    # Show the figure
    plt.show()


draw_graph_louvain_pr(G, pr_result=pr_result, louvain_result=louvain_result)
```

效果如图：

![draw_graph_louvain_pr](https://github.com/siwei-io/talks/assets/1651790/4d13eb2b-3e40-4085-8e0c-71459ca0843b)

更详细的例子参考：[ng_ai/examples](https://github.com/wey-gu/nebulagraph-ai/blob/main/examples/ng_ai_networkx_plot.ipynb)

#### 更方便的 Notebook 操作 NebulaGraph

结合 NebulaGraph 的 Jupyter Notebook 插件: https://github.com/wey-gu/ipython-ngql ，我们还可以更方便的操作 NebulaGraph：

在 Jupyter Notbook 里安装这个插件可以通过 ng_ai 的 extras 安装：

```python
%pip install ng_ai[jupyter]
%load_ext ngql
```

也可以单独安装
   
```python
%pip install ipython-ngql
%load_ext ngql
```

之后，我们就可以在 Notebook 里直接使用 `%ngql` 命令来执行 NGQL 语句了：

```python
%ngql --address 127.0.0.1 --port 9669 --user root --password nebula
%ngql USE basketballplayer;
%ngql MATCH (v:player{name:"Tim Duncan"})-->(v2:player) RETURN v2.player.name AS Name;
```

> 注，多行的 Query 用两个百分号就好了 `%%ngql`

最后，我们还能在 Jupyter Notebook 里直接可视化渲染结果！只需要 `%ng_draw` 就可以啦！
   
```python
%ngql match p=(:player)-[]->() return p LIMIT 5
%ng_draw
```

效果如下：

![ipython-ngql](https://user-images.githubusercontent.com/1651790/236797874-c142cfd8-b0de-4689-95ef-1da386e137d3.png)

## 未来工作

现在 ng_ai 还在开发中，我们还有很多工作要做：

- [ ] 完善 reader 模式，现在 NebulaGraph/NetworkX 的读取数据只支持 Query-Mode，还需要支持 Scan-Mode
- [ ] 实现基于 dgl(GNN) 的链路预测、节点分类等算法，例如：

```python
model = g.algo.gnn_link_prediction()
result = model.train()
# query src, dst to be predicted

model.predict(src_vertex, dst_vertices)
```

- [ ] UDA，自定义算法
- [ ] 快速部署工具

ng_ai 是完全 build in public 的，欢迎社区的大家们来参与，一起来完善 ng_ai，让 NebulaGraph 上的 AI 算法更加简单易用！

## 试玩 ng_ai

我们已经准备好了一键部署的 NebulaGraph + Studio + ng_ai in Jupyter 的环境，只需要大家从 Docker Desktop 的 Extension（扩展）中搜索 NebulaGraph，就可以试完了。

- 安装 [NebulaGraph Docker 插件](https://www.docker.com/blog/distributed-cloud-native-graph-database-nebulagraph-docker-extension/)

在 Docker Desktop 的插件市场搜索 NebulaGraph，点击安装

![](https://github.com/siwei-io/talks/assets/1651790/e07e39d1-2b3a-486f-bb68-29a67bc3a714)


- 安装 ng_ai playground

进入 NebulaGraph 插件，点击**Install NX Mode**，安装 ng_ai 的 NetworkX playground，通常要等几分钟等待安装完成。

![](https://github.com/siwei-io/talks/assets/1651790/3fca7e39-6e37-4ccb-8177-4771babb7b66)

- 进入 NetworkX playground

点击**Jupyter NB NetworkX**，进入 NetworkX playground。

![](https://github.com/siwei-io/talks/assets/1651790/48c2e5a2-490e-4626-809e-5894d6f833e9)

## ng_ai 的架构

ng_ai 的架构如下，它的核心模块有：

- Reader：负责从 NebulaGraph 读取数据
- Writer：负责将数据写入 NebulaGraph
- *Engine：负责适配不同运行时，例如 Spark、DGL、NetowrkX 等
- Algo：算法模块，例如 PageRank、Louvain、GNN_Link_Predict 等

此外，为了支持 nGQL 中的调用，还有两个模块：
- ng_ai-udf：负责将 UDF 注册到 NebulaGraph，接受 ng_ai 的 query 调用，访问 ng_ai API
- ng_ai-api：ng_ai 的 API 服务，接受 UDF 的调用，访问 ng_ai 核心模块

```asciiarmor
          ┌───────────────────────────────────────────────────┐
          │   Spark Cluster                                   │
          │    .─────.    .─────.    .─────.    .─────.       │
          │   ;       :  ;       :  ;       :  ;       :      │
       ┌─▶│   :       ;  :       ;  :       ;  :       ;      │
       │  │    ╲     ╱    ╲     ╱    ╲     ╱    ╲     ╱       │
       │  │     `───'      `───'      `───'      `───'        │
  Algo Spark                                                  │
    Engine└───────────────────────────────────────────────────┘
       │  ┌────────────────────────────────────────────────────┬──────────┐
       └──┤                                                    │          │
          │   NebulaGraph AI Suite(ngai)                       │ ngai-api │◀─┐
          │                                                    │          │  │
          │                                                    └──────────┤  │
          │     ┌────────┐    ┌──────┐    ┌────────┐   ┌─────┐            │  │
          │     │ Reader │    │ Algo │    │ Writer │   │ GNN │            │  │
 ┌───────▶│     └────────┘    └──────┘    └────────┘   └─────┘            │  │
 │        │          │            │            │          │               │  │
 │        │          ├────────────┴───┬────────┴─────┐    └──────┐        │  │
 │        │          ▼                ▼              ▼           ▼        │  │
 │        │   ┌─────────────┐ ┌──────────────┐ ┌──────────┐ ┌──────────┐  │  │
 │     ┌──┤   │ SparkEngine │ │ NebulaEngine │ │ NetworkX │ │ DGLEngine│  │  │
 │     │  │   └─────────────┘ └──────────────┘ └──────────┘ └──────────┘  │  │
 │     │  └──────────┬────────────────────────────────────────────────────┘  │
 │     │             │        Spark                                          │
 │     │             └────────Reader ────────────┐                           │
 │  Spark                   Query Mode           │                           │
 │  Reader                                       │                           │
 │Scan Mode                                      ▼                      ┌─────────┐
 │     │  ┌───────────────────────────────────────────────────┬─────────┤ ngai-udf│◀─────────────┐
 │     │  │                                                   │         └─────────┤              │
 │     │  │  NebulaGraph Graph Engine         Nebula-GraphD   │   ngai-GraphD     │              │
 │     │  ├──────────────────────────────┬────────────────────┼───────────────────┘              │
 │     │  │                              │                    │                                  │
 │     │  │  NebulaGraph Storage Engine  │                    │                                  │
 │     │  │                              │                    │                                  │
 │     └─▶│  Nebula-StorageD             │    Nebula-Metad    │                                  │
 │        │                              │                    │                                  │
 │        └──────────────────────────────┴────────────────────┘                                  │
 │                                                                                               │
 │    ┌───────────────────────────────────────────────────────────────────────────────────────┐  │
 │    │ RETURN ng_ai("pagerank", ["follow"], ["degree"], "spark", {space:"basketballplayer"}) │──┘
 │    └───────────────────────────────────────────────────────────────────────────────────────┘
 │  ┌─────────────────────────────────────────────────────────────┐
 │  │ from ng_ai import NebulaReader                              │
 │  │                                                             │
 │  │ # read data with spark engine, scan mode                    │
 │  │ reader = NebulaReader(engine="spark")                       │
 │  │ reader.scan(edge="follow", props="degree")                  │
 └──│ df = reader.read()                                          │
    │                                                             │
    │ # run pagerank algorithm                                    │
    │ pr_result = df.algo.pagerank(reset_prob=0.15, max_iter=10)  │
    │                                                             │
    └─────────────────────────────────────────────────────────────┘  
```


