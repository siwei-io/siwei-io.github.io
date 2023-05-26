# Nebulagraph Artificial Intelligence Suite


> Introducing a new project! ng_ai: NebulaGraph's graph algorithm suite, a user-friendly high-level Python Algorithm API for NebulaGraph. Its goal is to enable data scientist users of NebulaGraph to perform graph-related algorithmic tasks with minimal code.

<!--more-->


## Nebulagraph AI Suite

This week, NebulaGraph 3.5.0 [has been released](https://docs.nebula-graph.io/3.5.0/20.appendix/release-notes/nebula-comm-release-note/), and @**[whitewum](https://github.com/whitewum)** suggested that we make the new project [ng_ai](https://github.com/wey-gu/nebulagraph-ai) that has been launched in the NebulaGraph community public. This blog is the first one to introduce ng_ai!


### What is ng_ai

Nebulagraph AI Suite. As the name suggests, it is a Python suite for running algorithms on NebulaGraph. Its goal is to provide data scientist users of NebulaGraph with a natural, concise high-level API to perform graph-related algorithmic tasks with minimal code.


### Features

> Simplifying things in surprising ways.


To provide a smooth algorithmic experience for NebulaGraph community users, ng_ai has the following features:

- Tight integration with NebulaGraph
- Support for multiple engines and backends, currently supporting Spark (NebulaGraph Algorithm) and NetworkX, with plans to support DGL and PyG in the future.
- User-friendly and intuitive API design.
- Seamless integration with NebulaGraph's UDF, allowing ng_ai tasks to be called from queries.
- Friendly custom algorithm interface, making it easy for users to implement their own algorithms (WIP).
- One-click playground setup (based on Docker Extension).

## Demos

### Run PageRank

We could run distributed PageRank with Nebula-Algorithms(spark) backend:

```python
from ng_ai import NebulaReader

# read data with spark engine, scan mode
reader = NebulaReader(engine="spark")
reader.scan(edge="follow", props="degree")
df = reader.read()

# run pagerank algorithm
pr_result = df.algo.pagerank(reset_prob=0.15, max_iter=10)
```

### Write Algo Result to NebulaGraph

Assuming we want to run a label propagation algorithm and write the results back to NebulaGraph, we can do the following:

First, make sure that the schema of the TAG to be written back has been created, and write it to the label_propagation.cluster_id field:


```sql
CREATE TAG IF NOT EXISTS label_propagation (
    cluster_id string NOT NULL
);
```

The algorithm is run as follows:
   
```python
df_result = df.algo.label_propagation()

```

We could see its schema:

```python
df_result.printSchema()

# result
root
 |-- _id: string (nullable = false)
 |-- lpa: string (nullable = false)
```

Then, we write the result back to the cluster_id field in NebulaGraph (`{"lpa": "cluster_id"}`):

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

Finally, we can verify the results:

```cypher
USE basketballplayer;
MATCH (v:label_propagation)
RETURN id(v), v.label_propagation.cluster_id LIMIT 3;
```

The results are as follows:

```SQL
+-------------+--------------------------------+
| id(v)       | v.label_propagation.cluster_id |
+-------------+--------------------------------+
| "player103" | "player101"                    |
| "player113" | "player129"                    |
| "player121" | "player129"                    |
+-------------+--------------------------------+
```

See more examples: [ng_ai/examples](https://github.com/wey-gu/nebulagraph-ai/blob/main/examples/spark_engine.ipynb)

#### Call ng_ai from nGQL UDF

Since NebulaGraph 3.5.0, we can write our own UDF to call our own functions from nGQL. ng_ai also uses this capability to implement an ng_ai function that can call ng_ai algorithms from nGQL, for example:

```sql
-- Prepare the write schema
USE basketballplayer;
CREATE TAG IF NOT EXISTS pagerank(pagerank string);
:sleep 20;
-- Call with ng_ai()
RETURN ng_ai("pagerank", ["follow"], ["degree"], "spark", {space: "basketballplayer", max_iter: 10}, {write_mode: "insert"})
```

See more examples: [ng_ai/examples](https://github.com/wey-gu/nebulagraph-ai/blob/main/examples/ng_ai_from_ngql_udf.ipynb)

#### Example with NetworkX Engine

In a local environment, ng_ai supports running algorithms based on NetworkX, for example:

Read the graph as an ng_ai object:

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

Show and draw the graph:

```python
g.show(5)
g.draw()
```

Run PageRank:

```python
pr_result = g.algo.pagerank(reset_prob=0.15, max_iter=10)
```

Write the result back to NebulaGraph:

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

Other algorithms are similar, for example:

```python
# get all algorithms
g.algo.get_all_algo()

# get help of each algo
help(g.algo.node2vec)

# call the algo
g.algo.node2vec()
```

See more examples: [ng_ai/examples](https://github.com/wey-gu/nebulagraph-ai/blob/main/examples/networkx_engine.ipynb)

#### Plotting with NetworkX Engine

We could also run Louvain and PageRank and visualize the results with NetworkX engine:

First, we read the graph and run the algorithm:


```python
pr_result = g.algo.pagerank(reset_prob=0.15, max_iter=10)
louvain_result = g.algo.louvain()
```

Now we create a fancy function to draw the graph:

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

And it will show the graph:

![draw_graph_louvain_pr](https://github.com/siwei-io/talks/assets/1651790/4d13eb2b-3e40-4085-8e0c-71459ca0843b)

See more examples: [ng_ai/examples](https://github.com/wey-gu/nebulagraph-ai/blob/main/examples/ng_ai_networkx_plot.ipynb)

#### NebulaGraph Jupyter Notebook Extension

Thanks to the NebulaGraph Jupyter Notebook extension: https://github.com/wey-gu/ipython-ngql, we can operate NebulaGraph more conveniently:


Option 0, install the extension from ng_ai:

```python
%pip install ng_ai[jupyter]
%load_ext ngql
```

Option 1, install the extension directly:
   
```python
%pip install ipython-ngql
%load_ext ngql
```

Then, we can use `%ngql` to execute NGQL statements in the Notebook:

```python
%ngql --address 127.0.0.1 --port 9669 --user root --password nebula
%ngql USE basketballplayer;
%ngql MATCH (v:player{name:"Tim Duncan"})-->(v2:player) RETURN v2.player.name AS Name;
```

> Note, for multi-line query, use `%%ngql`

Finally, we can visualize the results in Jupyter Notebook with `%ng_draw`!
   
```python
%ngql match p=(:player)-[]->() return p LIMIT 5
%ng_draw
```

And it will look like this:

![ipython-ngql](https://user-images.githubusercontent.com/1651790/236797874-c142cfd8-b0de-4689-95ef-1da386e137d3.png)

## Future Work

Now ng_ai is still under development, we still have a lot of work to do:

- [ ] Improve the reader mode, now NebulaGraph/NetworkX only supports Query-Mode, we also need to support Scan-Mode
- [ ] Implement link prediction, node classification and other algorithms based on dgl (GNN), for example:

```python
model = g.algo.gnn_link_prediction()
result = model.train()
# query src, dst to be predicted

model.predict(src_vertex, dst_vertices)
```

- [ ] UDA, custom algorithm
- [ ] Deployment tool

ng_ai is completely built in public, and we welcome everyone in the community to participate in it and improve ng_ai together, making AI algorithms on NebulaGraph easier to use!

## Try ng_ai

We have prepared a one-click deployment of NebulaGraph + Studio + ng_ai in Jupyter environment, you only need to search NebulaGraph from the Extension of Docker Desktop to try it out.

- Install [NebulaGraph Docker Extension](https://www.docker.com/blog/distributed-cloud-native-graph-database-nebulagraph-docker-extension/)

Search NebulaGraph from docker extension marketplace, and install it.

![](https://github.com/siwei-io/talks/assets/1651790/e07e39d1-2b3a-486f-bb68-29a67bc3a714)


- Install ng_ai playground

Go to NebulaGraph extension, click **Install NX Mode** to install ng_ai's NetworkX playground, it usually takes a few minutes to wait for the installation to complete.

![](https://github.com/siwei-io/talks/assets/1651790/3fca7e39-6e37-4ccb-8177-4771babb7b66)

- Enter NetworkX playground

Click **Jupyter NB NetworkX** to enter NetworkX playground.

![](https://github.com/siwei-io/talks/assets/1651790/48c2e5a2-490e-4626-809e-5894d6f833e9)

## ng_ai Architecture

ng_ai is a Python library, it is mainly composed of the following modules:

- Writer: responsible for writing data to NebulaGraph
- Engine: responsible for adapting different runtimes, such as Spark, DGL, NetowrkX, etc.
- Algo: algorithm module, such as PageRank, Louvain, GNN_Link_Predict, etc.

In addition, in order to support the call in nGQL, there are two more modules:
- ng_ai-udf: responsible for registering UDF to NebulaGraph, accepting query calls from ng_ai, and accessing ng_ai API
- ng_ai-api: the API module of ng_ai, which is responsible for receiving requests from ng_ai-udf and calling the corresponding algorithm module

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

