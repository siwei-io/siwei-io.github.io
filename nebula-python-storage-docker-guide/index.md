# 快速搭建调试 Nebula Graph Python Storage 客户端的环境


> Python Storage Client 连不上第一次安装的 Nebula Graph？ Docker 部署情况下使用 Python Storage Client 指南

<!--more-->

## 前置条件

> 注意：一个重要的前置条件是我们真的需要 Storage Client，如果我们只需要在 Python Client 里通过 nGQL 来请求数据，那么 GraphClient 才是你需要的，可以跳过本文。

对于刚接触 Nebula 的同学，部署集群最快速的方式是用 Docker Compose，安装部署可以参考文档：[deploy-nebula-graph-with-docker-compose](https://docs.nebula-graph.com.cn/2.6.1/4.deployment-and-installation/2.compile-and-install-nebula-graph/3.deploy-nebula-graph-with-docker-compose/)。

在 `docker compose up -d` 之后，我们的 nebula 集群就起来了，我们可以用 `docker ps` 看看运行着的容器：

```bash
❯ docker ps --format "table {{.Names}}\t{{.Ports}}"
NAMES                               PORTS
nebula-docker-compose-graphd1-1     0.0.0.0:61852->9669/tcp, 0.0.0.0:61850->19669/tcp, 0.0.0.0:61851->19670/tcp
nebula-docker-compose-graphd-1      0.0.0.0:9669->9669/tcp, 0.0.0.0:61858->19669/tcp, 0.0.0.0:61859->19670/tcp
nebula-docker-compose-graphd2-1     0.0.0.0:61855->9669/tcp, 0.0.0.0:61853->19669/tcp, 0.0.0.0:61854->19670/tcp
nebula-docker-compose-storaged2-1   9777-9778/tcp, 9780/tcp, 0.0.0.0:61868->9779/tcp, 0.0.0.0:61869->19779/tcp, 0.0.0.0:61870->19780/tcp
nebula-docker-compose-storaged1-1   9777-9778/tcp, 9780/tcp, 0.0.0.0:61865->9779/tcp, 0.0.0.0:61866->19779/tcp, 0.0.0.0:61864->19780/tcp
nebula-docker-compose-storaged0-1   9777-9778/tcp, 9780/tcp, 0.0.0.0:61845->9779/tcp, 0.0.0.0:61843->19779/tcp, 0.0.0.0:61844->19780/tcp
nebula-docker-compose-metad1-1      9560/tcp, 0.0.0.0:61705->9559/tcp, 0.0.0.0:61706->19559/tcp, 0.0.0.0:61707->19560/tcp
nebula-docker-compose-metad2-1      9560/tcp, 0.0.0.0:61699->9559/tcp, 0.0.0.0:61700->19559/tcp, 0.0.0.0:61701->19560/tcp
nebula-docker-compose-metad0-1      9560/tcp, 0.0.0.0:61704->9559/tcp, 0.0.0.0:61702->19559/tcp, 0.0.0.0:61703->19560/tcp
```

这里边有三种容器 ：

- `GrpahD`，查询引擎，也是我们用户进行登录、连接、发 Query 请求直接访问的唯一一种服务、接口。
- `MetaD`，元数据服务，它一般不会暴露给外部，只有 GraphD 、StorageD 会直接访问它。
- `StorageD`，存储引擎，它一般不会暴露给外部，只有 GraphD、StorageD 会直接访问它。

我们可以从 Stuido Console，或者 Nebula Console （连接到 GraphD 之后）里通过 `SHOW HOSTS <Type>`，来获取每种服务的信息，其中第一列的信息就是他们的 IP 或者 Host，下边的例子是 Docker Compose 默认配置下的情况。

```sh
(root@nebula) [(none)]> SHOW HOSTS GRAPH
+-----------+------+----------+---------+--------------+---------+
| Host      | Port | Status   | Role    | Git Info Sha | Version |
+-----------+------+----------+---------+--------------+---------+
| "graphd"  | 9669 | "ONLINE" | "GRAPH" | "3ba41bd"    | "2.6.0" |
+-----------+------+----------+---------+--------------+---------+
| "graphd1" | 9669 | "ONLINE" | "GRAPH" | "3ba41bd"    | "2.6.0" |
+-----------+------+----------+---------+--------------+---------+
| "graphd2" | 9669 | "ONLINE" | "GRAPH" | "3ba41bd"    | "2.6.0" |
+-----------+------+----------+---------+--------------+---------+

(root@nebula) [(none)]> SHOW HOSTS META
+----------+------+----------+--------+--------------+---------+
| Host     | Port | Status   | Role   | Git Info Sha | Version |
+----------+------+----------+--------+--------------+---------+
| "metad1" | 9559 | "ONLINE" | "META" | "3ba41bd"    | "2.6.0" |
+----------+------+----------+--------+--------------+---------+
| "metad0" | 9559 | "ONLINE" | "META" | "3ba41bd"    | "2.6.0" |
+----------+------+----------+--------+--------------+---------+
| "metad2" | 9559 | "ONLINE" | "META" | "3ba41bd"    | "2.6.0" |
+----------+------+----------+--------+--------------+---------+

(root@nebula) [(none)]> SHOW HOSTS STORAGE
+-------------+------+----------+-----------+--------------+---------+
| Host        | Port | Status   | Role      | Git Info Sha | Version |
+-------------+------+----------+-----------+--------------+---------+
| "storaged0" | 9779 | "ONLINE" | "STORAGE" | "3ba41bd"    | "2.6.0" |
+-------------+------+----------+-----------+--------------+---------+
| "storaged1" | 9779 | "ONLINE" | "STORAGE" | "3ba41bd"    | "2.6.0" |
+-------------+------+----------+-----------+--------------+---------+
| "storaged2" | 9779 | "ONLINE" | "STORAGE" | "3ba41bd"    | "2.6.0" |
+-------------+------+----------+-----------+--------------+---------+
```

在默认的 [nebula-docker-compose](https://github.com/vesoft-inc/nebula-docker-compose) 的配置下各个服务为`graphd`, `metad1`, `storaged0` 这种域名的 Host 格式，这其实是假设了我们使用 Docker 启动的服务一般是作为单机测试之用，除了其中的 `graphd`之外，其他的服务都没有指定固定的外部映射端口（见前边的 `docker ps`  的结果里，它暴露在`0.0.0.0:9669`)。

这意味着，如果客户端不是运行在本机，访问其他服务的端口都是动态的，这会让很多第一次想用 Python Storage Client 连接服务的同学卡住。

所以我这里给大家分享一个快速用 Python 去调试 Storage Client 的方法：

> 本质上我们可以通过修改 Compose 的配置文件、通过其他部署或者配置的方式安装 Nebula 来保证 Python Client 能够访问 Storage/Meta 服务，这里只是给出一个无需修改配置的方法：让 Python 运行环境处在 Docker Compose 的 Nebula Cluster 同一个容器网络。

### 第一步，把 Python 容器

在 `nebula-docker-compose_nebula-net` 这个容器网络启动一个 Jupyter 的容器。

```bash
docker run \
    -p 8888:8888 \
    --network nebula-docker-compose_nebula-net \
    jupyter/scipy-notebook:33add21fab64
```

> 可以看到容器了，并且在监听端口：8888。

```bash
[I 07:06:31.060 NotebookApp] Jupyter Notebook 6.3.0 is running at:
[I 07:06:31.060 NotebookApp] http://e170a5eb4858:8888/?token=5beafb26fc6995b081c611d5d2cc96d557897b74bfdaac53
[I 07:06:31.060 NotebookApp] or http://127.0.0.1:8888/?token=5beafb26fc6995b081c611d5d2cc96d557897b74bfdaac53
```

这时候我们可在浏览器打开 http://127.0.0.1:8888/?token=5beafb26fc6995b081c611d5d2cc96d557897b74bfdaac53.

在里边新建一个 Notebook。

### 第二步，安装 Nebula Python SDK

我们只需要在 Jupyter 里执行

```python
!pip install nebula2-python==2.6.0
```

就可以了，具体的版本要根据 Nebula Python 的 README 里的[版本对应](https://github.com/vesoft-inc/nebula-python#how-to-choose-nebula-python)关系来给定。

### 第三步，实例化 MetaCache 和 GraphStorageClient

```python
from nebula2.mclient import MetaCache, HostAddr
from nebula2.sclient.GraphStorageClient import GraphStorageClient
# Docker Compose 下默认 meta 的地址是 metad1 metad0 metad2
meta_cache = MetaCache([('metad1',9559),('metad0',9559),('metad2',9559)])

graph_storage_client = GraphStorageClient(meta_cache)
```

### 第四步，扫数据

- 按空间、点类型扫点

```python
resp = graph_storage_client.scan_vertex(
        space_name='basketballplayer',
        tag_name='player')

while resp.has_next():
    result = resp.next()
    for vertex_data in result:
        print(vertex_data)
```

- 按空间、边类型扫边

```python
resp = graph_storage_client.scan_edge(
    space_name='basketballplayer',
    edge_name='follow')

while resp.has_next():
    result = resp.next()
    for edge_data in result:
        print(edge_data)
```

Jupyter 的过程我也记录在[这个 notebook](https://siwei.io/nebula-python-storage-docker-guide/Storage_Client_Example.ipynb) 里方便大家参考。

Happy Graphing!

> Picture Credit：[Borderpolar Photographer](https://unsplash.com/photos/AMXFr97d00c) 

