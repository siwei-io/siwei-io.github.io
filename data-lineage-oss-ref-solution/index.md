# 基于开源技术栈的数据血缘、治理参考解决方案


> 也许我们没有必要从头在 NebulaGraph 上搭建自己的数据血缘项目，本文分享如何用开源、现代的 DataOps、ETL、Dashboard、元数据、数据血缘管理系统构建大数据治理基础设施

<!--more-->

## 元数据治理系统

**元数据治理**系统是一个提供了所有数据在哪、它们的格式化方式、生成、转换、依赖、呈现和所属的**一站式视图**。

元数据治理系统是所有数据仓库、数据库、表、仪表板、ETL 作业等的**目录接口**（catalog），有了它，我们就不用在群里喊“大家好，我可以更改这个表的 schema 吗？”， “请问谁知道我如何找到 table-view-foo-bar 的原始数据？”，一个成熟的数据治理方案中的元数据治理系统，对成规模的数据团队来说非常必要。

对于另一个词：**数据血缘**则是众多需要管理的元数据之一，例如，某些 Dashboard 是 某一个 Table View 的下游，而这个 Table View 又是从另外两个上游的表 JOIN 而来两。 我们显然应该清晰的掌握、管理这些信息，去构建一个可信、可控的系统和数据质量控制体系。

## 参考解决方案

### 方案的动机

元数据和数据血缘本质上非常适合图数据建模、图数据库的场景。这里典型的查询就是面向图关系的查询了，像“查找每个给定组件（即表）的所有 n 深度数据血缘”就是一个 NebulaGraph 中的`FIND ALL PATH` 查询。

作为 NebulaGraph 社区中的一员，我发现人们在论坛、群里讨论的查询和图建模总能看出来很多人都在 NebulaGraph 上从头搭建自己的数据血缘系统，而这些工作看起来大多数都是重复造轮子（而且轮子并不容易造）。

我们来看看这样的元数据治理系统的轮子里，都需要那些功能组件：

- 元数据 extractor
  - 这部分需要从数据栈的不同方（如数据库、数仓、Dashboard，甚至从 ETL Pipeline 和应用、服务等等）中以拉或者推的方式获取。
- 元数据存储
  - 可以存在数据库、图数据库里，或者有时候存成超大的 JSON manifest 文件都行
- 元数据目录接口系统（Catalog）
  - 提供 API 和/或 GUI 界面以读取/写入元数据和数据血缘的系统

在 NebulaGraph 社区中，我看到不少人因为提问的查询和建模中明显有数据血缘的痕迹，意识到大家都在从头搭建数据血缘系统。考虑到系统中元数据的提取对象都是从各种知名数据库、数仓、最终的需求也大相径庭，这种重复的开发、研究、探索是一种大大的浪费。

所以，我准备搭建一个能够启发大家的参考数据血缘、治理方案，利用到市面上最好的开源项目。希望能让打算在 NebulaGraph 上定义和迭代自己的 Graph Model 并创建内部元数据和 pipeline 的人可以从这个项目中受益，从而拥有一个相对完善、设计精美的开箱即用的元数据治理系统，和相对更完善的图模型。

我尽量把这个方案做的完备、端到端（不只有元数据管理），希望也能为考虑做基于图做数据治理的新手一些启发和参考。

下图是整个方案的简单示意图：

其中上方是元数据的来源与导入、下方是元数据的存储与展示、发现。

![diagram-of-ref-project](https://user-images.githubusercontent.com/1651790/168849779-4826f50e-ff87-4e78-b17f-076f91182c43.svg)

### 技术栈介绍

下边介绍一下其中的每一部分。



#### 数据库和数仓

为了处理和使用原始和中间数据，这里一定涉及至少一个数据库或者数仓。

它可以是 Hive、Apache Delta、TiDB、Cassandra、MySQL 或 Postgres，在这个参考项目中，我们选一个简单、流行的 Postgres。

✅ - 数据仓库：Postgres

#### 数据运维 DataOps

我们应该有某种 DataOps 的方案，让 Pipeline 和环境具有可重复性、可测试性和版本控制性。

在这里，我们使用了 GitLab 创建的 [Meltano](https://gitlab.com/meltano/meltano)。

Meltano 是一个 just-work 的 DataOps 平台，它以一种神奇而优雅的方式将 [Singer](https://singer.io/) 作为 EL 和 [dbt](https://getdbt.com/) 作为 T 连接起来，它还连接到其他一些 dataInfra 实用程序，例如 Apache Superset 和 Apache Airflow 等。

至此，我们又纳入了一个成员：

✅ - GitOps：Meltano https://gitlab.com/meltano/meltano

#### ETL

如前边提到，我们还利用 [Singer](https://singer.io/) 与 Meltano 一起将来自许多不同数据源的数据 E（提取）和 L（加载）数据目标，并使用 [dbt](https://getdbt.com/) 作为 Transform 的平台。

✅ - EL：Singer https://singer.io/

✅ - T: dbt https://getdbt.com/

#### 数据可视化

在数据之上创建 Dashboard、图表和表格来获得洞察是很直接的需求（可以想象为想象大数据之上的 excel 图标功能）。

![](https://user-images.githubusercontent.com/1651790/172800854-8e01acae-696d-4e07-8e3e-a7d34dec8278.png)

[Apache Superset](https://superset.apache.org/) 是我很喜欢的开源数据可视化项目，我准备用它来作为被治理管理的目标之一，同时，也会利用它的可视化作为元数据洞察功能的一部分。

✅ - Dashboard：Apache Superset https://superset.apache.org/

#### 任务编排（DAG Job Orchestration）

在大多数情况下，我们的 DataOps 作业、任务会增长到需要一个编排系统的规模，我们可以用 [Apache Airflow](https://airflow.apache.org/) 来负责这一块。

✅ - DAG：Apache Airflow https://airflow.apache.org/

#### 元数据治理

随着越来越多的组件和数据被引入数据基础设施，在数据库、表、数据建模(schema)、Dashboard、DAG（编排系统中的有向无环图）、应用与服务的所有生命周期中都将存在海量的元数据，需要对它们的管理员和团队进行协同管理、连接和发现。

[Linux Foundation Amundsen](https://www.amundsen.io/amundsen/) 是我认为可以解决这个问题的最佳项目之一。

![](https://user-images.githubusercontent.com/1651790/172801018-ecd67fa9-2743-451f-8734-b14f2a814199.png)



✅ - 数据发现：Linux Foundation Amundsen https://www.amundsen.io/amundsen/

Amundsen 用图数据库为事实源（single source of truth）以加速多跳查询，Elastic Search 为全文搜索引擎，它能对所有元数据及其血缘进行了顺滑的处理还提供了优雅的 UI 和 API。

Amundsen 支持多种图数据库为后端，这里咱们用 [NebulaGraph](https://nebula-graph.com.cn)。

✅ - 全文搜索：Elastic Search

✅ - 图数据库：NebulaGraph

现在，所有组件都齐活了，开始组装它们吧。

## 环境搭建与各组件初识

整个项目方案都是开源的，大家可以在这里找到它的所有细节：

- https://github.com/wey-gu/data-lineage-ref-solution

整个项目大家的实验中我遵循尽量干净、鼓励的原则，需要假设在一个 unix-like 的系统上运行，有互联网和 Docker-Compose。

> 注：参考 https://docs.docker.com/compose/install/ 在继续之前安装 Docker 和 Docker Compose。

这里我们在 Ubuntu 20.04 LTS X86_64 上运行它，但在其他发行版或 Linux 版本上应该也没有问题。

### 运行一个数仓、数据库

首先，安装 Postgres 作为我们的数仓。

这个单行命令会创建一个使用 docker 在后台运行的 Postgres，进程关闭之后容器不会残留而是被清理掉（因为参数`--rm`）。

```bash
docker run --rm --name postgres \
    -e POSTGRES_PASSWORD=lineage_ref \
    -e POSTGRES_USER=lineage_ref \
    -e POSTGRES_DB=warehouse -d \
    -p 5432:5432 postgres
```

Then we could verify it with Postgres CLI or GUI clients.

> Hint: You could use VS Code extension: [SQL tools](https://marketplace.visualstudio.com/items?itemName=mtxr.sqltools) to quickly connect to multiple RDBMS(MariaDB, Postgres, etc.) or even Non-SQL DBMS like Cassandra in a GUI fashion.

然后我们可以使用 Postgres CLI 或 GUI 客户端来验证它。

> 提示：可以用 VS Code 插件：[SQL TOOLS](https://marketplace.visualstudio.com/items?itemName=mtxr.sqltools) 快速以 GUI 方式连接到数据库（支持 MariaDB、Postgres 、Cassandra 等）
>
> https://marketplace.visualstudio.com/items?itemName=mtxr.sqltools

### DataOps 工具链部署

然后，安装有机结合了 Singler 和 dbt 的 Meltano。

Meltano 帮助我们管理 ETL 工具（作为插件）及其所有配置和 pipeline。 这些元信息位于 meltano 配置及其系统数据库（https://docs.meltano.com/concepts/project#system-database）中，其中配置是基于文件的（可以使用 GitOps 管理），它的默认系统数据库是 SQLite。

#### 安装 Meltano

使用 Meltano 的工作流是启动一个“meltano 项目”并开始将 E、L 和 T 添加到配置文件中。 项目的启动只需要一个 CLI 命令调用：`meltano init yourprojectname`，在那之前，可以先用 Python 的包管理器：pip 或者 Docker 镜像安装 Meltano：

- 在 python 虚拟环境中使用 pip 安装 Meltano：

```bash
mkdir .venv
# example in a debian flavor Linux distro
sudo apt-get install python3-dev python3-pip python3-venv python3-wheel -y
python3 -m venv .venv/meltano
source .venv/meltano/bin/activate
python3 -m pip install wheel
python3 -m pip install meltano

# init a project
mkdir meltano_projects && cd meltano_projects
# replace <yourprojectname> with your own one
touch .env
meltano init <yourprojectname>
```

- 或者用容器安装 Meltano：

```bash
docker pull meltano/meltano:latest
docker run --rm meltano/meltano --version

# init a project
mkdir meltano_projects && cd meltano_projects

# replace <yourprojectname> with your own one
touch .env
docker run --rm -v "$(pwd)":/projects \
             -w /projects --env-file .env \
             meltano/meltano init <yourprojectname>
```

除了 `meltano init`，还有一些其他命令，例如 `meltano etl` 表示 ETL 的执行，还有 `meltano invoke <plugin>` 来调用插件命令，详细可以参考它的速查表（https://docs.meltano.com/reference/command-line-interface）。

#### Meltano GUI 界面

Meltano 还带有一个基于 Web 的 UI，执行 `ui` 子命令就是启动它：

```bash
meltano ui
```

默认他会跑在 http://localhost:5000 上。

对于 Docker 运行的情况，只需要在暴露 5000 端口的情况下运行容器即可，由于容器的默认命令已经是 `meltano ui`，所以 `run` 的命令只需：

```bash
docker run -v "$(pwd)":/project \
             -w /project \
             -p 5000:5000 \
             meltano/meltano
```

#### Meltano 项目示例

写到这里的时候，我注意到 [Pat Nadolny](https://github.com/pnadolny13) 创建了很好的示例项目在 https://github.com/pnadolny13/meltano_example_implementations/tree/main/meltano_projects/singer_dbt_jaffle，它利用 dbt 的 Meltano 示例数据集，采用 Airflow 编排 ETL 任务（https://github.com/pnadolny13/meltano_example_implementations/tree/main/meltano_projects/dbt_orchestration，还有利用 Superset 的例子（https://github.com/pnadolny13/meltano_example_implementations/tree/main/meltano_projects/jaffle_superset）。

这里，我就不重复造轮子了，直接利用他的例子吧。

咱们可以参照 https://github.com/pnadolny13/meltano_example_implementations/tree/main/meltano_projects/singer_dbt_jaffle，运行这样的数据管道（pipeline）：

- [tap-CSV](https://hub.meltano.com/taps/csv)(Singer)，从 CSV 文件中提取数据
- [target-postgres](https://hub.meltano.com/targets/postgres)(Singer)，将数据加载到 Postgres
- [dbt](https://hub.meltano.com/transformers/dbt)，将数据转换为聚合表或视图

> 注意，前边我们已经启动了 postgres，那一步可以跳过。

操作过程是：

```bash
git clone https://github.com/pnadolny13/meltano_example_implementations.git
cd meltano_example_implementations/meltano_projects/singer_dbt_jaffle/

meltano install
touch .env
echo PG_PASSWORD="lineage_ref" >> .env
echo PG_USERNAME="lineage_ref" >> .env

# Extract and Load(with Singer)
meltano run tap-csv target-postgres

# Trasnform(with dbt)
meltano run dbt:run

# Generate dbt docs
meltano invoke dbt docs generate

# Serve generated dbt docs
meltano invoke dbt docs to serve

# Then visit http://localhost:8080
```

现在，我们可以连接到 Postgres 来查看 加载和转换后的数据预览如下，截图来自 VS Code 的 SQLTool：

Payments 表里长这样子：

![](https://user-images.githubusercontent.com/1651790/167540494-01e3dbd2-6ab1-41d2-998e-3b79f755bdc7.png)

### 搭一个 BI Dashboard 系统

现在，我们有了数据仓库中的一些数据，用 ETL 工具链将不同的数据源导了进去，接下来可以试着用一下这些数据了。

像仪表大盘 Dashbaord 这样的 BI 工具能帮助我们从数据中获得有用的洞察，使用 Apache Superset，可以很容易地创建和管理基于这些数据源的 Dashboard 和各式各样的图表。

本章的重点不在于 Apache Superset 本身，所以，咱们还是复用 [Pat Nadolny](https://github.com/pnadolny13) 在的例子 https://github.com/pnadolny13/meltano_example_implementations/tree/main/meltano_projects/jaffle_superset。

#### Bootstrap Meltano 和 Superset

创建一个安装了 Meltano 的 python venv：

```bash
mkdir .venv
python3 -m venv .venv/meltano
source .venv/meltano/bin/activate
python3 -m pip install wheel
python3 -m pip install meltano
```

参考 Pat 的 Guide（https://github.com/pnadolny13/meltano_example_implementations/tree/main/meltano_projects/jaffle_superset），稍微做一些修改：

- 克隆 repo，进入 `jaffle_superset` 项目

```bash
git clone https://github.com/pnadolny13/meltano_example_implementations.git
cd meltano_example_implementations/meltano_projects/jaffle_superset/
```

- 修改meltano配置文件，让 Superset 连接到我们创建的 Postgres：

```bash
vim meltano_projects/jaffle_superset/meltano.yml
```

这里，我将主机名更改为“10.1.1.111”，这是我当前主机的 IP，而如果读者在 Windows 或者 macOS 机器的 Docker Desktop 上跑的话，这里不要修改，否则要参考我去改成自己实际的地址：

```diff
--- a/meltano_projects/jaffle_superset/meltano.yml
+++ b/meltano_projects/jaffle_superset/meltano.yml
@@ -71,7 +71,7 @@ plugins:
               A list of database driver dependencies can be found here https://superset.apache.org/docs/databases/installing-database-drivers
     config:
       database_name: my_postgres
-      sqlalchemy_uri: postgresql+psycopg2://${PG_USERNAME}:${PG_PASSWORD}@host.docker.internal:${PG_PORT}/${PG_DATABASE}
+      sqlalchemy_uri: postgresql+psycopg2://${PG_USERNAME}:${PG_PASSWORD}@10.1.1.168:${PG_PORT}/${PG_DATABASE}
       tables:
       - model.my_meltano_project.customers
       - model.my_meltano_project.orders
```

- 添加 Postgres 登录的信息到  `.env` 文件：

```bash
echo PG_USERNAME=lineage_ref >> .env
echo PG_PASSWORD=lineage_ref >> .env
```

- 安装 Meltano 项目，运行 ETL 任务

```bash
meltano install
meltano run tap-csv target-postgres dbt:run
```

- 调用、启动 superset，这里注意 `ui` 不是 meltano 的内部命令，而是一个配置进去的自定义行为（user-defined action）

```bash
meltano invoke superset:ui
```

- 在另一个命令行终端，执行另一个自定义的命令 `load_datasources`

```
meltano invoke superset:load_datasources
```

- 通过浏览器访问 http://localhost:8088/ 就是Superset 的图形界面了：

![](https://user-images.githubusercontent.com/1651790/168570300-186b56a5-58e8-4ff1-bc06-89fd77d74166.png)

#### 创建一个 Dashboard

试一下在这个 Meltano 项目中定义的 Postgres 中的 ETL 数据上创建一个 Dashboard 吧

- 点击 `+ DASHBOARD`，填写仪表盘名称，然后点击 `SAVE`，然后点击 `+ CREATE A NEW CHART`

![](https://user-images.githubusercontent.com/1651790/168570363-c6b4f929-2aad-4f03-8e3e-b1b61f560ce5.png)

- 在新图表（Create a new chart）视图中，我们应该选择图表类型和数据集。 在这里，我选择了 `orders` 表作为数据源和 `Pie Chart` 图表类型：

![](https://user-images.githubusercontent.com/1651790/168570927-9559a2a1-fed7-43be-9f6a-f6fb3c263830.png)

- 点击“CREATE NEW CHART”后，我们在图表定义视图中，我选择了“status”的“Query”为“DIMENSIONS”，“COUNT(amount)”为“METRIC”。 至此，咱们就可以看到每个订单状态分布的饼图了。

![](https://user-images.githubusercontent.com/1651790/168571130-a65ba88e-1ebe-4699-8783-08e5ecf54a0c.png)

- 点击 `SAVE` ，它会询问应该将此图表添加到哪个 Dashboard，选择后，单击 `SAVE & GO TO DASHBOARD`。

![](https://user-images.githubusercontent.com/1651790/168571301-8ae69983-eda8-4e75-99cf-6904f583fc7c.png)

- 然后，在 Dashboard 中，我们可以看到那里的所有图表。 您可以看到我还添加了另一个图表来显示客户订单数量分布：

![](https://user-images.githubusercontent.com/1651790/168571878-30a77057-1f66-448a-9bbd-0dedcee24cc9.png)

- 点  `···` 的话，还能看到刷新率设置、下载渲染图等其他的功能。

![](https://user-images.githubusercontent.com/1651790/168573874-b5d57919-2866-4b3c-a4e5-55b6e6ef342e.png)

目前，我们有一个简单但典型的 homelab 数据技术栈了，并且所有东西都是开源的！

想象一下，我们在 CSV 中有 100 个数据集，在数据仓库中有 200 个表，并且有几个数据工程师在运行不同的项目，这些项目使用、生成不同的应用与服务、Dashbaord 和数据库。 当有人想要查找、发现或者修改其中的一些表、数据集、Dashbaord 和管道，在沟通和工程方面可能都是非常不好管理的。

如前边提到的，我们需要这个示例项目的主要部分：元数据发现系统。

### 元数据发现系统

然后，我们部署一个带有 NebulaGraph 和 Elasticsearch 的 Amundsen。

> 注：目前【NebulaGraph 作为 Amundsen 后端的 PR】（https://github.com/amundsen-io/amundsen/pull/1817）尚未合并，我还在与 Amundsen 团队合作（https://github.com/amundsen-io/rfcs/pull/48）来实现它。

有了 Amundsen，我们可以在一个地方发现和管理整个数据栈中的所有元数据。 

Amundsen 主要有两个部分组成：

- 元数据导入 Metadata Ingestion
  - [Amundsen Data builder](https://www.amundsen.io/amundsen/databuilder/)
- 元数据目录服务 Metadata Catalog
  - [Amundsen Frontend service](https://www.amundsen.io/amundsen/frontend/)
  - [Amundsen Metadata service](https://www.amundsen.io/amundsen/metadata/)
  - [Amundsen Search service](https://www.amundsen.io/amundsen/search/)

它的工作原理是：利用 `Data builder` 从不同来源提取元数据，并将元数据持久化到 `Meta service` 的后端存储和 `Search service` 的后端存储中，用户从 `Froent service` 或通过 `Meta Service` 的API。

#### 部署 Amundsen

##### 元数据服务 Metadata service

我们用 docker-compose 文件部署一个 Amundsen 集群。 由于 NebulaGraph 后端支持尚未合并，还不能用官方的代码，先用我自己的分叉版本。

首先，让我们克隆包含所有子模块的 repo：

```bash
git clone -b amundsen_nebula_graph --recursive git@github.com:wey-gu/amundsen.git
cd amundsen
```

然后，启动所有目录服务（catalog services）及其后端存储：

```bash
docker-compose -f docker-amundsen-nebula.yml up
```

> 注：可以添加 `-d` 来让容器在后台运行：
>
> ```bash
> docker-compose -f docker-amundsen-nebula.yml up -d
> ```
>
> 关闭后台运行的集群
>
> ```bash
> docker-compose -f docker-amundsen-nebula.yml stop
> ```
>
> 删除后台运行的集群
>
> ```bash
> docker-compose -f docker-amundsen-nebula.yml down
> ```

由于这个 docker-compose 文件是供开发人员试玩、调试 Amundsen 用的，而不是给生产部署准备的，它在启动的时候会从代码库构建镜像，第一次跑的时候启动会慢一些。

部署好了之后，我们使用 Data builder 将一些示例、虚构的数据加载存储里。

##### 抓取元数据 Data builder

Amundsen Data builder 就像 Meltano 系统一样，只不过是用在元数据的上的 ETL ，它把元数据加载到“Meta service”和“Search service”的后端存储：NebulaGraph 和 Elasticsearch 里。 这里的 Data builder 只是一个 python 模块，所有的元数据 ETL 作业可以作为脚本运行，也可以用 Apache Airflow 等 DAG 平台进行编排。

安装 [Amundsen Data builder](https://github.com/amundsen-io/amundsen/tree/main/databuilder)：

```bash
cd databuilder
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install wheel
python3 -m pip install -r requirements.txt
python3 setup.py install
```

调用这个示例数据构建器 ETL 脚本来把示例的虚拟数据导进去。

```bash
python3 example/scripts/sample_data_loader_nebula.py
```

##### 验证一下 Amundsen

在访问 Amundsen 之前，我们需要创建一个测试用户：

```bash
# run a container with curl attached to amundsenfrontend
docker run -it --rm --net container:amundsenfrontend nicolaka/netshoot

# Create a user with id test_user_id
curl -X PUT -v http://amundsenmetadata:5002/user \
    -H "Content-Type: application/json" \
    --data \
    '{"user_id":"test_user_id","first_name":"test","last_name":"user", "email":"test_user_id@mail.com"}'

exit
```

然后我们可以在 [http://localhost:5000](http://localhost:5000/) 查看 UI 并尝试搜索 `test`，它应该会返回一些结果。

![](https://github.com/amundsen-io/amundsen/raw/master/docs/img/search-page.png)

然后，可以单击并浏览在“sample_data_loader_nebula.py”期间加载到 Amundsen 的那些示例元数据。

此外，我们还可以通过 NebulaStudio(http://localhost:7001) 访问 NebulaGraph 里的这些数据。

> 注意在 Nebula Studio 中，默认登录字段为：
>
> - 主机：`graphd:9669`
> - 用户：`root`
> - 密码：`nebula`

下图显示了有关 Amundsen 组件的更多详细信息：

```asciiarmor
       ┌────────────────────────┐ ┌────────────────────────────────────────┐
       │ Frontend:5000          │ │ Metadata Sources                       │
       ├────────────────────────┤ │ ┌────────┐ ┌─────────┐ ┌─────────────┐ │
       │ Metaservice:5001       │ │ │        │ │         │ │             │ │
       │ ┌──────────────┐       │ │ │ Foo DB │ │ Bar App │ │ X Dashboard │ │
  ┌────┼─┤ Nebula Proxy │       │ │ │        │ │         │ │             │ │
  │    │ └──────────────┘       │ │ │        │ │         │ │             │ │
  │    ├────────────────────────┤ │ └────────┘ └─────┬───┘ └─────────────┘ │
┌─┼────┤ Search searvice:5002   │ │                  │                     │
│ │    └────────────────────────┘ └──────────────────┼─────────────────────┘
│ │    ┌─────────────────────────────────────────────┼───────────────────────┐
│ │    │                                             │                       │
│ │    │ Databuilder     ┌───────────────────────────┘                       │
│ │    │                 │                                                   │
│ │    │ ┌───────────────▼────────────────┐ ┌──────────────────────────────┐ │
│ │ ┌──┼─► Extractor of Sources           ├─► nebula_search_data_extractor │ │
│ │ │  │ └───────────────┬────────────────┘ └──────────────┬───────────────┘ │
│ │ │  │ ┌───────────────▼────────────────┐ ┌──────────────▼───────────────┐ │
│ │ │  │ │ Loader filesystem_csv_nebula   │ │ Loader Elastic FS loader     │ │
│ │ │  │ └───────────────┬────────────────┘ └──────────────┬───────────────┘ │
│ │ │  │ ┌───────────────▼────────────────┐ ┌──────────────▼───────────────┐ │
│ │ │  │ │ Publisher nebula_csv_publisher │ │ Publisher Elasticsearch      │ │
│ │ │  │ └───────────────┬────────────────┘ └──────────────┬───────────────┘ │
│ │ │  └─────────────────┼─────────────────────────────────┼─────────────────┘
│ │ └────────────────┐   │                                 │
│ │    ┌─────────────┼───►─────────────────────────┐ ┌─────▼─────┐
│ │    │ Nebula Graph│   │                         │ │           │
│ └────┼─────┬───────┴───┼───────────┐     ┌─────┐ │ │           │
│      │     │           │           │     │MetaD│ │ │           │
│      │ ┌───▼──┐    ┌───▼──┐    ┌───▼──┐  └─────┘ │ │           │
│ ┌────┼─►GraphD│    │GraphD│    │GraphD│          │ │           │
│ │    │ └──────┘    └──────┘    └──────┘  ┌─────┐ │ │           │
│ │    │ :9669                             │MetaD│ │ │  Elastic  │
│ │    │ ┌────────┐ ┌────────┐ ┌────────┐  └─────┘ │ │  Search   │
│ │    │ │        │ │        │ │        │          │ │  Cluster  │
│ │    │ │StorageD│ │StorageD│ │StorageD│  ┌─────┐ │ │  :9200    │
│ │    │ │        │ │        │ │        │  │MetaD│ │ │           │
│ │    │ └────────┘ └────────┘ └────────┘  └─────┘ │ │           │
│ │    ├───────────────────────────────────────────┤ │           │
│ └────┤ Nebula Studio:7001                        │ │           │
│      └───────────────────────────────────────────┘ └─────▲─────┘
└──────────────────────────────────────────────────────────┘
```



## 穿针引线：元数据发现

设置好基本环境后，让我们把所有东西穿起来。还记得我们有 ELT 一些数据到 PostgreSQL 吗？

![](https://user-images.githubusercontent.com/1651790/167540494-01e3dbd2-6ab1-41d2-998e-3b79f755bdc7.png)

那么，我们如何让 Amundsen 发现有关这些数据和 ETL 的元数据呢？

### 提取 Postgres 元数据

我们从数据源开始：首先是 Postgres。

我们为 python3 安装 Postgres 客户端：

```bash
sudo apt-get install libpq-dev
pip3 install Psycopg2
```

#### 执行 Postgres 元数据 ETL

运行一个脚本来解析 Postgres 元数据：

```bash
export CREDENTIALS_POSTGRES_USER=lineage_ref
export CREDENTIALS_POSTGRES_PASSWORD=lineage_ref
export CREDENTIALS_POSTGRES_DATABASE=warehouse

python3 example/scripts/sample_postgres_loader_nebula.py
```

If you look into the code of the sample script for loading Postgres metadata to Nebula, the main lines are quite straightforward:

我们看看把 Postgres 元数据加载到 NebulaGraph 的示例脚本的代码，非常简单直接：

```python
# part 1: PostgressMetadata --> CSV --> Nebula Graph
job = DefaultJob(
      conf=job_config,
      task=DefaultTask(
          extractor=PostgresMetadataExtractor(),
          loader=FsNebulaCSVLoader()),
      publisher=NebulaCsvPublisher())

...
# part 2: Metadata stored in NebulaGraph --> Elasticsearch
extractor = NebulaSearchDataExtractor()
task = SearchMetadatatoElasticasearchTask(extractor=extractor)

job = DefaultJob(conf=job_config, task=task)
```

第一个工作路径是：`PostgressMetadata --> CSV --> Nebula Graph`

- `PostgresMetadataExtractor` 用于从 Postgres 中提取/提取元数据，可以参考文档（https://www.amundsen.io/amundsen/databuilder/#postgresmetadataextractor）。
- `FsNebulaCSVLoader` 用于将提取的数据中间放置为 CSV 文件
- `NebulaCsvPublisher` 用于将元数据以 CSV 的形式发布到 NebulaGraph

第二个工作路径是：`Metadata stored in NebulaGraph --> Elasticsearch`

- `NebulaSearchDataExtractor` 用于获取存储在 Nebula Graph 中的元数据
- `SearchMetadatatoElasticasearchTask` 用于使 Elasticsearch 对元数据进行索引。

> 请注意，在生产环境中，我们可以在脚本中或使用 Apache Airflow 等编排平台触发这些作业。

#### 验证 Postgres 中元数据的获取

搜索`payments`或者直接访问http://localhost:5000/table_detail/warehouse/postgres/public/payments，你可以看到我们 Postgres 的元数据，比如：

![](https://user-images.githubusercontent.com/1651790/168475180-ebfaa188-268c-4fbe-a614-135d56d07e5d.png)

然后，像上面的屏幕截图一样，可以轻松完成元数据管理操作，如添加标签、所有者和描述。

### 提取 dbt 元数据

实际上，我们也可以从 [dbt](https://www.getdbt.com/) 本身提取元数据。

Amundsen [DbtExtractor](https://www.amundsen.io/amundsen/databuilder/#dbtextractor) 会解析 `catalog.json` 或 `manifest.json` 文件以将元数据加载到 Amundsen 存储（NebulaGraph 和 Elasticsearch )。

在上面的 meltano 章节中，我们已经使用 `meltano invoke dbt docs generate` 生成了这个文件：

```log
14:23:15  Done.
14:23:15  Building catalog
14:23:15  Catalog written to /home/ubuntu/ref-data-lineage/meltano_example_implementations/meltano_projects/singer_dbt_jaffle/.meltano/transformers/dbt/target/catalog.json
```

#### dbt 元数据 ETL 的执行

我们试着解析示例 dbt 文件中的元数据吧：

```bash
$ ls -l example/sample_data/dbt/
total 184
-rw-rw-r-- 1 w w   5320 May 15 07:17 catalog.json
-rw-rw-r-- 1 w w 177163 May 15 07:17 manifest.json
```

我写的这个示例的加载例子如下：

```bash
python3 example/scripts/sample_dbt_loader_nebula.py
```

其中主要的代码如下：

```python
# part 1: Dbt manifest --> CSV --> Nebula Graph
job = DefaultJob(
      conf=job_config,
      task=DefaultTask(
          extractor=DbtExtractor(),
          loader=FsNebulaCSVLoader()),
      publisher=NebulaCsvPublisher())

...
# part 2: Metadata stored in NebulaGraph --> Elasticsearch
extractor = NebulaSearchDataExtractor()
task = SearchMetadatatoElasticasearchTask(extractor=extractor)

job = DefaultJob(conf=job_config, task=task)
```

它和 Postgres 元数据 ETL 的唯一区别是 `extractor=DbtExtractor()`，它带有以下配置以获取有关 dbt 项目的以下信息：

- 数据库名称
- 目录_json
- manifest_json

```python
job_config = ConfigFactory.from_dict({
  'extractor.dbt.database_name': database_name,
  'extractor.dbt.catalog_json': catalog_file_loc,  # File
  'extractor.dbt.manifest_json': json.dumps(manifest_data),  # JSON Dumped objecy
  'extractor.dbt.source_url': source_url})
```

#### 验证 dbt 抓取结果

搜索 `dbt_demo` 或者直接访问 http://localhost:5000/table_detail/dbt_demo/snowflake/public/raw_inventory_value，可以看到

![](https://user-images.githubusercontent.com/1651790/168479864-2f73ea73-265f-4cd2-999f-e7effbaf3ec1.png)

> 小提示：我们可以选择启用 DEBUG log 级别去看已发送到 Elasticsearch 和 NebulaGraph 的内容。
>
> ```diff
> - logging.basicConfig(level=logging.INFO)
> + logging.basicConfig(level=logging.DEBUG)
> ```

或者，在 NebulaStudio 中探索导入的数据：

首先，点击 “Start with Vertices”，填写顶点 vid：`snowflake://dbt_demo.public/fact_warehouse_inventory`

![](https://user-images.githubusercontent.com/1651790/168480047-26c28cde-5df8-40af-8da4-6ab0203094e2.png)



然后，我们可以看到顶点显示为粉红色的点。 让我们修改 `Expand` / ”拓展“选项：

- 方向：双向
- 步数：单向、三步

![](https://user-images.githubusercontent.com/1651790/168480101-7b7b5824-06d9-4155-87c9-798db0dc7612.png)

并双击顶点（点），它将双向拓展 3 步：

![](https://user-images.githubusercontent.com/1651790/168480280-1dc88d1b-1f1e-48fd-9997-972965522ef5.png)



从上边这个截图里我们可以发现，在可视化之后的图数据库中，这些元数据可以很容易被查看、分析，并从中获得洞察。

> 小贴士，您可以点击 👁 图标选择一些要显示的属性，我在截图之前就是通过它让一些信息显示出来的。

而且，我们在 NebulaStudio 中看到的也与 Amundsen 元数据服务的数据模型相呼应：

![](https://www.amundsen.io/amundsen/img/graph_model.png)

最后，请记住我们曾利用 dbt 来转换meltano 中的一些数据，并且清单文件路径是`.meltano/transformers/dbt/target/catalog.json`，您可以尝试创建一个数据构建器作业来导入它。

### 提取 Superset 中的元数据

Amundsen 的 Superset extractor 可以获取

- Dashboard 元数据抽取 https://www.amundsen.io/amundsen/databuilder/databuilder/extractor/dashboard/apache_superset/apache_superset_metadata_extractor.py
- 图表元数据抽取 https://www.amundsen.io/amundsen/databuilder/databuilder /extractor/dashboard/apache_superset/apache_superset_chart_extractor.py
- Superset 元素与数据源（表）的关系抽取 https://www.amundsen.io/amundsen/databuilder/databuilder/extractor/dashboard/apache_superset/apache_superset_table_extractor.py

咱们现在就尝试摄取之前创建的 Superset Dashboard 的元数据。

#### Superset 元数据 ETL 的执行

下边执行的示例 Superset 提取脚本可以从中获取数据并将元数据加载到 NebulaGraph 和 Elasticsearch 中。

```python
python3 sample_superset_data_loader_nebula.py
```

如果我们将日志记录级别设置为“DEBUG”，我们实际上可以看到这些中间的过程日志：

```python
# fetching metadata from superset
DEBUG:urllib3.connectionpool:http://localhost:8088 "POST /api/v1/security/login HTTP/1.1" 200 280
INFO:databuilder.task.task:Running a task
DEBUG:urllib3.connectionpool:Starting new HTTP connection (1): localhost:8088
DEBUG:urllib3.connectionpool:http://localhost:8088 "GET /api/v1/dashboard?q=(page_size:20,page:0,order_direction:desc) HTTP/1.1" 308 374
DEBUG:urllib3.connectionpool:http://localhost:8088 "GET /api/v1/dashboard/?q=(page_size:20,page:0,order_direction:desc) HTTP/1.1" 200 1058
...

# insert Dashboard

DEBUG:databuilder.publisher.nebula_csv_publisher:Query: INSERT VERTEX `Dashboard` (`dashboard_url`, `name`, published_tag, publisher_last_updated_epoch_ms) VALUES  "superset_dashboard://my_cluster.1/3":("http://localhost:8088/superset/dashboard/3/","my_dashboard","unique_tag",timestamp());
...

# insert a DASHBOARD_WITH_TABLE relationship/edge

INFO:databuilder.publisher.nebula_csv_publisher:Importing data in edge files: ['/tmp/amundsen/dashboard/relationships/Dashboard_Table_DASHBOARD_WITH_TABLE.csv']
DEBUG:databuilder.publisher.nebula_csv_publisher:Query:
INSERT edge `DASHBOARD_WITH_TABLE` (`END_LABEL`, `START_LABEL`, published_tag, publisher_last_updated_epoch_ms) VALUES "superset_dashboard://my_cluster.1/3"->"postgresql+psycopg2://my_cluster.warehouse/orders":("Table","Dashboard","unique_tag", timestamp()), "superset_dashboard://my_cluster.1/3"->"postgresql+psycopg2://my_cluster.warehouse/customers":("Table","Dashboard","unique_tag", timestamp());
```

#### 验证 Superset Dashboard 元数据

通过在 Amundsen 中搜索它，我们现在可以获得 Dashboard 信息。 

我们也可以从 NebulaStudio 进行验证。

![](https://user-images.githubusercontent.com/1651790/168719624-738323dd-4c6e-475f-a370-f149181c6184.png)

> 注：可以参阅 [Dashboard 抓取指南](https://www.amundsen.io/amundsen/databuilder/docs/dashboard_ingestion_guide/) 中的 Amundsen Dashboard 图建模：
>
> ![dashboard_graph_modeling](https://www.amundsen.io/amundsen/databuilder/docs/assets/dashboard_graph_modeling.png?raw=true)

### 用 Superset 预览数据

Superset could be used to preview Table Data like this. Corresponding documentation could be referred [here](https://www.amundsen.io/amundsen/frontend/docs/configuration/#preview-client), where the API of `/superset/sql_json/` will be called by Amundsen Frontend.

Superset可以用来预览这样的表格数据。 相应的文档可以参考 https://www.amundsen.io/amundsen/frontend/docs/configuration/#preview-client ，其中 `/superset/sql_json/` 的 API 被 Amundsen Frontend service 调用，取得预览信息。

![](https://github.com/amundsen-io/amundsenfrontendlibrary/blob/master/docs/img/data_preview.png?raw=true)

### 开启数据血缘信息

默认情况下，数据血缘是关闭的，我们可以通过以下方式启用它：

0. `cd` 到 Amundsen 代码仓库下，这也是我们运行 `docker-compose -f docker-amundsen-nebula.yml up` 命令的地方

```bash
cd amundsen
```

1. 修改 frontend 下的 typescript 配置

```diff
--- a/frontend/amundsen_application/static/js/config/config-default.ts
+++ b/frontend/amundsen_application/static/js/config/config-default.ts
   tableLineage: {
-    inAppListEnabled: false,
-    inAppPageEnabled: false,
+    inAppListEnabled: true,
+    inAppPageEnabled: true,
     externalEnabled: false,
     iconPath: 'PATH_TO_ICON',
     isBeta: false,
```

2. 重新构建 docker 镜像，其中将重建前端图像。

```bash
docker-compose -f docker-amundsen-nebula.yml build
```

然后，重新运行 `up -d` 以确保前端用新的配置：

```bash
docker-compose -f docker-amundsen-nebula.yml up -d
```

结果大概长这样子：

```bash
$ docker-compose -f docker-amundsen-nebula.yml up -d
...
Recreating amundsenfrontend           ... done
```

之后，我们可以访问 http://localhost:5000/lineage/table/gold/hive/test_schema/test_table1 看到 `Lineage （beta）` 血缘按钮已经显示出来了：

![](https://user-images.githubusercontent.com/1651790/168838731-79d0e3bc-439e-4f6b-8ef7-83b37e9bcb12.png)

我们可以点击 `Downstream` 在存在的时候查看该表的下游资源：

![](https://user-images.githubusercontent.com/1651790/168839251-efd523af-d729-44cf-a40b-fa83a0852654.png)

或者点血缘按钮查看血缘的图表式：

![](https://user-images.githubusercontent.com/1651790/168838814-e6ff5152-c24b-470e-a46a-48f183ba7201.png)

也有用于血缘查询的 API。 这个例子中我们用 cURL 调用下这个 API：

```bash
docker run -it --rm --net container:amundsenfrontend nicolaka/netshoot

curl "http://amundsenmetadata:5002/table/snowflake://dbt_demo.public/raw_inventory_value/lineage?depth=3&direction=both"
```

上面的 API 调用是查询上游和下游方向的 linage，表 `snowflake://dbt_demo.public/raw_inventory_value` 的深度为 3。

结果应该是这样的：

```json
{
    "depth": 3,
    "downstream_entities": [
        {
            "level": 2,
            "usage": 0,
            "key": "snowflake://dbt_demo.public/fact_daily_expenses",
            "parent": "snowflake://dbt_demo.public/fact_warehouse_inventory",
            "badges": [],
            "source": "snowflake"
        },
        {
            "level": 1,
            "usage": 0,
            "key": "snowflake://dbt_demo.public/fact_warehouse_inventory",
            "parent": "snowflake://dbt_demo.public/raw_inventory_value",
            "badges": [],
            "source": "snowflake"
        }
    ],
    "key": "snowflake://dbt_demo.public/raw_inventory_value",
    "direction": "both",
    "upstream_entities": []
}
```

实际上，这个血缘数据就是在我们的 [DbtExtractor](https://github.com/amundsen-io/amundsen/blob/main/databuilder/databuilder/extractor/dbt_extractor.py) 执行期间提取和加载的，其中 `extractor .dbt.{DbtExtractor.EXTRACT_LINEAGE}` 默认为 `True`，因此创建了血缘元数据并将其加载到了 Amundsen。

#### 在 NebulaGraph 中洞察血缘

使用图数据库作为元数据存储的两个优点是：

- 图查询本身是一个灵活的 DSL for lineage API，例如，这个查询帮助我们执行 Amundsen 元数据 API 的等价的查询：

```cypher
MATCH p=(t:`Table`) -[:`HAS_UPSTREAM`|:`HAS_DOWNSTREAM` *1..3]->(x)
WHERE id(t) == "snowflake://dbt_demo.public/raw_inventory_value" RETURN p
```

- 我们现在甚至可以在 NebulaGraph Studio 或者 Explorer 的控制台中查询它

![](https://user-images.githubusercontent.com/1651790/168844882-ca3d0587-7946-4e17-8264-9dc973a44673.png)

​    然后渲染这个结果：

![](https://user-images.githubusercontent.com/1651790/168845155-b0e7a5ce-3ddf-4cc9-89a3-aaf1bbb0f5ec.png)

#### 提取数据血缘

这些血缘信息是需要我们明确指定、获取的，获取的方式可以是自己写的 extractor，也可以是一些已经有的方式。比如 dbt 的 extractor和 Open Lineage 项目的 Amundsen extractor。

##### 通过 dbt

这个在刚才已经展示过了，Dbt 的 Extractor 会从表级别获取血缘和其他 dbt 中产生的元数据信息一起被拿到。

##### 通过 Open Lineage

Amundsen 中的另一个开箱即用的血缘 Extractor 是 [OpenLineageTableLineageExtractor](https://www.amundsen.io/amundsen/databuilder/#openlineagetablelineageextractor)。

[Open Lineage](https://openlineage.io/) 是一个开放的框架，可以将不同来源的血统数据收集到一个地方，它可以将血统信息输出为 JSON 文件：https://www.amundsen.io/amundsen/databuilder/#openlineagetablelineageextractor

下边是它的 Amundsen data builder 例子：

```python
dict_config = {
    # ...
    f'extractor.openlineage_tablelineage.{OpenLineageTableLineageExtractor.CLUSTER_NAME}': 'datalab',
    f'extractor.openlineage_tablelineage.{OpenLineageTableLineageExtractor.OL_DATASET_NAMESPACE_OVERRIDE}': 'hive_table',
    f'extractor.openlineage_tablelineage.{OpenLineageTableLineageExtractor.TABLE_LINEAGE_FILE_LOCATION}': 'input_dir/openlineage_nd.json',
}
...

task = DefaultTask(
    extractor=OpenLineageTableLineageExtractor(),
    loader=FsNebulaCSVLoader())
```

## 回顾

整套元数据治理/发现的方案思路如下：

- 将整个数据技术栈中的组件作为元数据源（从任何数据库、数仓，到 dbt、Airflow、Openlineage、Superset 等各级项目）
- 使用 Databuilder（作为脚本或 DAG）运行元数据 ETL，以使用 NebulaGraph 和 Elasticsearch 存储和索引
- 从前端 UI（使用 Superset 预览）或 API 去使用、消费、管理和发现元数据
- 通过查询和 UI 对 NebulaGraph，我们可以获得更多的可能性、灵活性和数据、血缘的洞察

![](https://user-images.githubusercontent.com/1651790/168849779-4826f50e-ff87-4e78-b17f-076f91182c43.svg)



### 涉及到的开源

此参考项目中使用的所有项目都按字典顺序在下面列出。

- Amundsen
- Apache Airflow
- Apache Superset
- dbt
- Elasticsearch
- meltano
- Nebula Graph
- Open Lineage
- singer

> 题图版权： [Phil Hearing](https://unsplash.com/photos/PhnJhjH9Y9s)

