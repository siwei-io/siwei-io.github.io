# Nebula LiveJournal, Import LiveJournal Dataset into Nebula Graph and Run Nebula Algorithm



> 一个导入 Livejournal 数据集到 Nebula Graph 图数据库，并执行 Nebula Algorithm 图算法的过程分享。

<!--more-->

Related GitHub Repo: https://github.com/wey-gu/nebula-LiveJournal



# nebula-LiveJournal

LiveJournal Dataset is a Social Network Dataset in one file with two columns(FromNodeId, ToNodeId).

```bash
$ head soc-LiveJournal1.txt
# Directed graph (each unordered pair of nodes is saved once): soc-LiveJournal1.txt
# Directed LiveJournal friednship social network
# Nodes: 4847571 Edges: 68993773
# FromNodeId	ToNodeId
0	1
0	2
0	3
0	4
0	5
0	6
```

It could be accessed in https://snap.stanford.edu/data/soc-LiveJournal1.html.

| Dataset statistics               |                  |
| :------------------------------- | ---------------- |
| Nodes                            | 4847571          |
| Edges                            | 68993773         |
| Nodes in largest WCC             | 4843953 (0.999)  |
| Edges in largest WCC             | 68983820 (1.000) |
| Nodes in largest SCC             | 3828682 (0.790)  |
| Edges in largest SCC             | 65825429 (0.954) |
| Average clustering coefficient   | 0.2742           |
| Number of triangles              | 285730264        |
| Fraction of closed triangles     | 0.04266          |
| Diameter (longest shortest path) | 16               |
| 90-percentile effective diameter | 6.5              |

## Dataset Download and Preprocessing

### Download

It is accesissiable from the official web page:

```bash
$ cd nebula-livejournal/data
$ wget https://snap.stanford.edu/data/soc-LiveJournal1.txt.gz
```

Comments in data file should be removed to make the data import tool happy.

### Preprocessing

```bash
$ gzip -d soc-LiveJournal1.txt.gz
$ sed -i '1,4d' soc-LiveJournal1.txt
```

## Import dataset to Nebula Graph

### With Nebula Importer

[Nebula-Importer](https://github.com/vesoft-inc/nebula-importer) is a Golang Headless import tool for Nebula Graph.

You may need to edit the config file under [nebula-importer/importer.yaml](nebula-importer/importer.yaml) on Nebula Graph's address and credential。

Then, Nebula-Importer could be called in Docker as follow:

```bash
$ cd nebula-livejournal

$ docker run --rm -ti \
    --network=nebula-net \
    -v nebula-importer/importer.yaml:/root/importer.yaml \
    -v data/:/root \
    vesoft/nebula-importer:v2 \
    --config /root/importer.yaml
```

Or if you have the binary nebula-importer locally:

```bash
$ cd data
$ <path_to_nebula-importer_binary> --config ../nebula-importer/importer.yaml
```



### With Nebula Exchange

[Nebula-Exchange](https://github.com/vesoft-inc/nebula-spark-utils/tree/master/nebula-exchange) is a Spark Application to enable batch and streaming data import from multiple data sources to Nebula Graph.

To be done. (You can refer to https://siwei.io/nebula-exchange-sst-2.x/)

## Run Algorithms with Nebula Graph

[Nebula-Algorithm](https://github.com/vesoft-inc/nebula-spark-utils/tree/master/nebula-algorithm) is a Spark/GraphX Application to run Graph Algorithms with data consumed from files or a Nebula Graph Cluster.

Supported Algorithms for now:

| Name                       | Use Case                                                     |
| -------------------------- | ------------------------------------------------------------ |
| PageRank                   | page ranking, important node digging                         |
| Louvain                    | community digging, hierarchical clustering                   |
| KCore                      | community detection, financial risk control                  |
| LabelPropagation           | community detection, consultation propagation, advertising recommendation |
| ConnectedComponent         | community detection, isolated island detection               |
| StronglyConnectedComponent | community detection                                          |
| ShortestPath               | path plan, network plan                                      |
| TriangleCount              | network structure analysis                                   |
| BetweennessCentrality      | important node digging, node influence calculation           |
| DegreeStatic               | graph structure analysis                                     |

### Ad-hoc Spark Env setup

Here I assume the Nebula Graph was bootstraped with [Nebula-Up](https://github.com/wey-gu/nebula-up), thus nebula is running in a Docker Network named `nebula-docker-compose_nebula-net`.

Then let's start a single server spark:

```bash
docker run --name spark-master --network nebula-docker-compose_nebula-net \
    -h spark-master -e ENABLE_INIT_DAEMON=false -d \
    -v nebula-algorithm/:/root \
    bde2020/spark-master:2.4.5-hadoop2.7
```

Thus we could make spark application submt inside this container:

```bash
docker exec -it spark-master bash
cd /root/
# download Nebula-Algorithm Jar Packagem, 2.0.0 for example, for other versions, refer to nebula-algorithm github repo and documentations.
wget https://repo1.maven.org/maven2/com/vesoft/nebula-algorithm/2.0.0/nebula-algorithm-2.0.0.jar
```

### Run Algorithms

There are many altorithms supported by Nebula-Algorithm, here some of their configuration files were put under nebula-algorithm as an example.

Before using them, please first edit and change Nebula Graph Cluster Addresses and credentials.

```bash
vim nebula-altorithm/algo-pagerank.conf
```

Then we could enter the spark container and call corresponding algorithms as follow.

Please adjust your `--driver-memeory` accordingly, i.e. pagerank altorithm:

```bash
/spark/bin/spark-submit --master "local" --conf spark.rpc.askTimeout=6000s \
    --class com.vesoft.nebula.algorithm.Main \
    --driver-memory 16g nebula-algorithm-2.0.0.jar \
    -p pagerank.conf
```

After the algorithm finished, the output will be under the path insdie the container defined in conf file:

```toml
    write:{
        resultPath:/output/
    }
```



> 题图版权：[@sigmund](https://unsplash.com/photos/eTgMFFzroGc) 


