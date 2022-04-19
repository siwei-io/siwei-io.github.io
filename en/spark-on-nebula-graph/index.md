# Spark on Nebula Graph


> What could be done with Spark and PySpark on top of Nebula Graph, this post covers everything we should know.

<!--more-->


In this article, I am trying walk you through all three Spark projects of Nebula Graph with some runnable hands-on examples. Also, I managed to make PySpark usable with Nebula Graph Spark Connector, which will be contributed to the Docs later.

## The three Spark projects for Nebula Graph

I used to draw a sketch around all data importing methods of Nebula Graph [here](https://www.siwei.io/sketches/nebula-data-import-options/), where all three of the Spark based Nebula Graph projects were already briefly introduced. In this article, instead, a slightly deeper dive on all of them will be made based on my recent work on them.

TL;DR

- Nebula Spark Connector is a Spark Lib to enable spark application reading from and writing to Nebula Graph in form of dataframe.
- Nebula Exchange, built on top of Nebula Spark Connector, is a Spark Lib and Application to exchange(for Open Source version, it's one way: write, whereas for enterprise version it's bidirectional) different data sources like(MySQL, Neo4j, PostgreSQL, Clickhouse, Hive etc.). Besides write directly to Nebula Graph, it could optionally generate SST files to be ingested into Nebula Graph to off load the storage computation outside of the Nebula Graph cluster.
- Nebula Algorithm, built on top of Nebula Spark Connector and GraphX, is an Spark Lib and Application to run de facto graph algorithms(pagerank, LPA etc...) on a graph from Nebula Graph.

Then let's have the long verson of those spark projects more on how-to perspectives.

## Spark-Connector

- Codebase: https://github.com/vesoft-inc/nebula-algorithm
- Documentation: https://docs.nebula-graph.io/3.0.2/nebula-algorithm/ (it's versioned, as for now, I put the lastest released version 3.0.2 here)
- Jar Package: https://repo1.maven.org/maven2/com/vesoft/nebula-algorithm/
- Code Examples: [example/src/main/scala/com/vesoft/nebula/algorithm](example/src/main/scala/com/vesoft/nebula/algorithm)



### Nebula Graph Spark Reader

To read data from Nebula Graph, i.e. vertex, Nebula Spark Connector will scan all storage instances who hold given label(TAG): `withLabel("player")`, and we could optionally specify the properties of the vertex: `withReturnCols(List("name", "age"))`.

With needed configuration being provided, a call of `spark.read.nebula.loadVerticesToDF` will return dataframe of the Vertex Scan call towards Nebula Graph: 

```scala
  def readVertex(spark: SparkSession): Unit = {
    LOG.info("start to read nebula vertices")
    val config =
      NebulaConnectionConfig
        .builder()
        .withMetaAddress("metad0:9559,metad1:9559,metad2:9559")
        .withConenctionRetry(2)
        .build()
    val nebulaReadVertexConfig: ReadNebulaConfig = ReadNebulaConfig
      .builder()
      .withSpace("basketballplayer")
      .withLabel("player")
      .withNoColumn(false)
      .withReturnCols(List("name", "age"))
      .withLimit(10)
      .withPartitionNum(10)
      .build()
    val vertex = spark.read.nebula(config, nebulaReadVertexConfig).loadVerticesToDF()
    vertex.printSchema()
    vertex.show(20)
    println("vertex count: " + vertex.count())
  }
```

It's similar for the writor part and one big difference here is the wrinting path is done via GraphD as underlying Spark Connector is shooting nGQL `INSERT` queries.

Then let's do a hands-on end to end practise.

### Hands-on Spark Connector

Prerequisites: it's assumed below procedure being run on a Linux Machine with internet connection, ideally with Docker and Docker-Compose preinstalled.

#### Bootstrap a Nebula Graph Cluster

Firstly, let's deploy Nebula Graph Core v3.0 and Nebula Studio with [Nebula-Up](https://github.com/wey-gu/nebula-up/), it will try to install Docker and Docker-Compose for us, in case it failed, please try install Docker and Docker-Compose on your own first.

```bash
curl -fsSL nebula-up.siwei.io/install.sh | bash -s -- v3.0
```

After the above script being executed, let's connect to it with Nebula-Console, the command line client for Nebula Graph.

- Enter the container with console

```bash
~/.nebula-up/console.sh
```

- Connect to the Nebula Graph

```bash
nebula-console -addr graphd -port 9669 -user root -p nebula
```

- Activate Storage Instances, and check hosts status

  > ref: https://docs.nebula-graph.io/3.0.2/4.deployment-and-installation/manage-storage-host/

```bash
ADD HOSTS "storaged0":9779,"storaged1":9779,"storaged2":9779;
SHOW HOSTS;
```

- Load the [test graph data](https://docs.nebula-graph.io/3.0.2/3.ngql-guide/1.nGQL-overview/1.overview/#example_data_basketballplayer), which will take one or two minitues to finish.

```bash
:play basketballplayer;
```

#### Create a Spark playground

Thanks to [Big data europe](https://github.com/big-data-europe/docker-spark), it's quite handly to do so:

```bash
docker run --name spark-master-0 --network nebula-docker-compose_nebula-net \
    -h spark-master-0 -e ENABLE_INIT_DAEMON=false -d \
    -v ${PWD}/:/root \
    bde2020/spark-master:2.4.5-hadoop2.7
```

In above one line command, we created a container named `spark-master-0` with a built-in hadoop 2.7 and spark 2.4.5, connected to the Nebula Graph cluster in its docker network named `nebula-docker-compose_nebula-net`, and it mapped the current path to `/root` of the spark container.

Then, we could access to the spark env container with:

```bash
docker exec -it spark-master-0 bash
```

Optionally, we could install `mvn` inside the container:

```bash
docker exec -it spark-master-0 bash
# in the container shell

export MAVEN_VERSION=3.5.4
export MAVEN_HOME=/usr/lib/mvn
export PATH=$MAVEN_HOME/bin:$PATH

wget http://archive.apache.org/dist/maven/maven-3/$MAVEN_VERSION/binaries/apache-maven-$MAVEN_VERSION-bin.tar.gz && \
  tar -zxvf apache-maven-$MAVEN_VERSION-bin.tar.gz && \
  rm apache-maven-$MAVEN_VERSION-bin.tar.gz && \
  mv apache-maven-$MAVEN_VERSION /usr/lib/mvn

which /usr/lib/mvn/bin/mvn
```

#### Run spark connector example

Let's clone the connector and the example code base, and build(or place the connector Jar package) the connector:

```bash
git clone https://github.com/vesoft-inc/nebula-spark-connector.git

docker exec -it spark-master-0 bash
cd /root/nebula-spark-connector

/usr/lib/mvn/bin/mvn install -Dgpg.skip -Dmaven.javadoc.skip=true -Dmaven.test.skip=true
```

Then we replace the example code:

```bash
vi example/src/main/scala/com/vesoft/nebula/examples/connector/NebulaSparkReaderExample.scala
```

We put the code as the following, where two functions `readVertex` and `readEdges` was created on the `basketballplayer` graph space:

```scala
package com.vesoft.nebula.examples.connector

import com.facebook.thrift.protocol.TCompactProtocol
import com.vesoft.nebula.connector.connector.NebulaDataFrameReader
import com.vesoft.nebula.connector.{NebulaConnectionConfig, ReadNebulaConfig}
import org.apache.spark.SparkConf
import org.apache.spark.sql.SparkSession
import org.slf4j.LoggerFactory

object NebulaSparkReaderExample {

  private val LOG = LoggerFactory.getLogger(this.getClass)

  def main(args: Array[String]): Unit = {

    val sparkConf = new SparkConf
    sparkConf
      .set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
      .registerKryoClasses(Array[Class[_]](classOf[TCompactProtocol]))
    val spark = SparkSession
      .builder()
      .master("local")
      .config(sparkConf)
      .getOrCreate()

    readVertex(spark)
    readEdges(spark)

    spark.close()
    sys.exit()
  }

  def readVertex(spark: SparkSession): Unit = {
    LOG.info("start to read nebula vertices")
    val config =
      NebulaConnectionConfig
        .builder()
        .withMetaAddress("metad0:9559,metad1:9559,metad2:9559")
        .withConenctionRetry(2)
        .build()
    val nebulaReadVertexConfig: ReadNebulaConfig = ReadNebulaConfig
      .builder()
      .withSpace("basketballplayer")
      .withLabel("player")
      .withNoColumn(false)
      .withReturnCols(List("name", "age"))
      .withLimit(10)
      .withPartitionNum(10)
      .build()
    val vertex = spark.read.nebula(config, nebulaReadVertexConfig).loadVerticesToDF()
    vertex.printSchema()
    vertex.show(20)
    println("vertex count: " + vertex.count())
  }

  def readEdges(spark: SparkSession): Unit = {
    LOG.info("start to read nebula edges")

    val config =
      NebulaConnectionConfig
        .builder()
        .withMetaAddress("metad0:9559,metad1:9559,metad2:9559")
        .withTimeout(6000)
        .withConenctionRetry(2)
        .build()
    val nebulaReadEdgeConfig: ReadNebulaConfig = ReadNebulaConfig
      .builder()
      .withSpace("basketballplayer")
      .withLabel("follow")
      .withNoColumn(false)
      .withReturnCols(List("degree"))
      .withLimit(10)
      .withPartitionNum(10)
      .build()
    val edge = spark.read.nebula(config, nebulaReadEdgeConfig).loadEdgesToDF()
    edge.printSchema()
    edge.show(20)
    println("edge count: " + edge.count())
  }

}
```

Then build it:

```bash
cd example
/usr/lib/mvn/bin/mvn install -Dgpg.skip -Dmaven.javadoc.skip=true -Dmaven.test.skip=true
```

Execute it on spark:

```bash
/spark/bin/spark-submit --master "local" \
    --class com.vesoft.nebula.examples.connector.NebulaSparkReaderExample \
    --driver-memory 16g target/example-3.0-SNAPSHOT.jar
```

And the result should include:

```
22/04/19 07:29:34 INFO DAGScheduler: Job 1 finished: show at NebulaSparkReaderExample.scala:57, took 0.199310 s
+---------+------------------+---+
|_vertexId|              name|age|
+---------+------------------+---+
|player105|       Danny Green| 31|
|player109|    Tiago Splitter| 34|
|player111|        David West| 38|
|player118| Russell Westbrook| 30|
|player143|Kristaps Porzingis| 23|
|player114|     Tracy McGrady| 39|
|player150|       Luka Doncic| 20|
|player103|          Rudy Gay| 32|
|player113|   Dejounte Murray| 29|
|player121|        Chris Paul| 33|
|player128|   Carmelo Anthony| 34|
|player130|       Joel Embiid| 25|
|player136|        Steve Nash| 45|
|player108|        Boris Diaw| 36|
|player122|    DeAndre Jordan| 30|
|player123|       Ricky Rubio| 28|
|player139|        Marc Gasol| 34|
|player142|     Klay Thompson| 29|
|player145|      JaVale McGee| 31|
|player102| LaMarcus Aldridge| 33|
+---------+------------------+---+
only showing top 20 rows

22/04/19 07:29:36 INFO DAGScheduler: Job 4 finished: show at NebulaSparkReaderExample.scala:82, took 0.135543 s
+---------+---------+-----+------+
|   _srcId|   _dstId|_rank|degree|
+---------+---------+-----+------+
|player105|player100|    0|    70|
|player105|player104|    0|    83|
|player105|player116|    0|    80|
|player109|player100|    0|    80|
|player109|player125|    0|    90|
|player118|player120|    0|    90|
|player118|player131|    0|    90|
|player143|player150|    0|    90|
|player114|player103|    0|    90|
|player114|player115|    0|    90|
|player114|player140|    0|    90|
|player150|player120|    0|    80|
|player150|player137|    0|    90|
|player150|player143|    0|    90|
|player103|player102|    0|    70|
|player113|player100|    0|    99|
|player113|player101|    0|    99|
|player113|player104|    0|    99|
|player113|player105|    0|    99|
|player113|player106|    0|    99|
+---------+---------+-----+------+
only showing top 20 rows
```

And actually there are more examples under the repo, especially one for GraphX, you could try exploring youself for that part. Please be noted in GraphX it assumed vertex ID to be in numeric type, thus for string typed vertex ID case a convertion on the fly is needed, please refer to [the example in Nebula Algorithom](https://github.com/vesoft-inc/nebula-algorithm/blob/a82d7092d928a2f3abc45a727c24afb888ff8e4f/example/src/main/scala/com/vesoft/nebula/algorithm/PageRankExample.scala#L31) on how to mitigate that.

## Exchange

- Codebase: https://github.com/vesoft-inc/nebula-exchange/
- Documentation: https://docs.nebula-graph.io/3.0.2/nebula-exchange/about-exchange/ex-ug-what-is-exchange/ (it's versioned, as for now, I put the lastest released version 3.0.2 here)
- Jar Package: https://github.com/vesoft-inc/nebula-exchange/releases
- Configuration Examples: [exchange-common/src/test/resources/application.conf](https://github.com/vesoft-inc/nebula-exchange/blob/master/exchange-common/src/test/resources/application.conf)

Nebula Exchange is a Spark Lib/App to read data from multiple sources, then, write to either Nebula Graph directly or into Nebula Graph [SST Files](https://docs.nebula-graph.io/3.0.2/nebula-exchange/use-exchange/ex-ug-import-from-sst/#step_5_import_the_sst_file).

![](https://docs.nebula-graph.io/3.0.2/nebula-exchange/figs/ex-ug-003.png)

The way to leverage Nebula Exchange is only to firstly create the configuration files to let exchange know how data should be fetched and written, then call the exchange package with the conf file specified.

Now let's do a hands on test with the same envrioment created in last chapter.

### Hands-on Exchange

Here, we are using Exchange to consume data source from a CSV file, where first column is Vertex ID, and second, third to be properties of "name" and "age".

```bash
player800,"Foo Bar",23
player801,"Another Name",21
```

- Let's get into the spark container created in last chapter, and download the Jar package of Nebula Exchange:

```bash
docker exec -it spark-master bash
cd /root/

wget https://github.com/vesoft-inc/nebula-exchange/releases/download/v3.0.0/nebula-exchange_spark_2.4-3.0.0.jar

```

- Create a conf file named `exchange.conf` in format `HOCON`, where:
  - under `.nebula`, information regarding Nebula Graph Cluster were configured
  - under `.tags`, information regarding Vertecies like how required fields are reflected to our data source(here it's CSV file) were configured

```HOCON
{
  # Spark relation config
  spark: {
    app: {
      name: Nebula Exchange
    }

    master:local

    driver: {
      cores: 1
      maxResultSize: 1G
    }

    executor: {
        memory: 1G
    }

    cores:{
      max: 16
    }
  }

  # Nebula Graph relation config
  nebula: {
    address:{
      graph:["graphd:9669"]
      meta:["metad0:9559", "metad1:9559", "metad2:9559"]
    }
    user: root
    pswd: nebula
    space: basketballplayer

    # parameters for SST import, not required
    path:{
        local:"/tmp"
        remote:"/sst"
        hdfs.namenode: "hdfs://localhost:9000"
    }

    # nebula client connection parameters
    connection {
      # socket connect & execute timeout, unit: millisecond
      timeout: 30000
    }

    error: {
      # max number of failures, if the number of failures is bigger than max, then exit the application.
      max: 32
      # failed import job will be recorded in output path
      output: /tmp/errors
    }

    # use google's RateLimiter to limit the requests send to NebulaGraph
    rate: {
      # the stable throughput of RateLimiter
      limit: 1024
      # Acquires a permit from RateLimiter, unit: MILLISECONDS
      # if it can't be obtained within the specified timeout, then give up the request.
      timeout: 1000
    }
  }

  # Processing tags
  # There are tag config examples for different dataSources.
  tags: [

    # HDFS csv
    # Import mode is client, just change type.sink to sst if you want to use client import mode.
    {
      name: player
      type: {
        source: csv
        sink: client
      }
      path: "file:///root/player.csv"
      # if your csv file has no header, then use _c0,_c1,_c2,.. to indicate fields
      fields: [_c1, _c2]
      nebula.fields: [name, age]
      vertex: {
        field:_c0
      }
      separator: ","
      header: false
      batch: 256
      partition: 32
    }

  ]
}
```

- Finally, let's create `player.csv` and `exchange.conf`, it should be listed as the following:

```bash
# ls -l

-rw-r--r--    1 root     root          1912 Apr 19 08:21 exchange.conf
-rw-r--r--    1 root     root     157814140 Apr 19 08:17 nebula-exchange_spark_2.4-3.0.0.jar
-rw-r--r--    1 root     root            52 Apr 19 08:06 player.csv
```

-  And we could call the exchange as:

```bash
/spark/bin/spark-submit --master local \
    --class com.vesoft.nebula.exchange.Exchange nebula-exchange_spark_2.4-3.0.0.jar \
    -c exchange.conf
```

And the result should be like

```log
...
22/04/19 08:22:08 INFO Exchange$: import for tag player cost time: 1.32 s
22/04/19 08:22:08 INFO Exchange$: Client-Import: batchSuccess.player: 2
22/04/19 08:22:08 INFO Exchange$: Client-Import: batchFailure.player: 0
...
```



Please refer to documentation and conf examples for more datasources. For hands on Exchange writing to SST Files, you could refer to both Documentation and [Nebula Exchange SST 2.x Hands-on Guide](https://www.siwei.io/nebula-exchange-sst-2.x/).



## Algorithm

- Codebase: https://github.com/vesoft-inc/nebula-exchange/
- Documentation: https://docs.nebula-graph.io/3.0.2/nebula-exchange/about-exchange/ex-ug-what-is-exchange/ (it's versioned, as for now, I put the lastest released version 3.0.2 here)
- Jar Package: https://github.com/vesoft-inc/nebula-exchange/releases
- Calling Algorithm in code examples: [exchange-common/src/test/resources/application.conf](https://github.com/vesoft-inc/nebula-exchange/blob/master/exchange-common/src/test/resources/application.conf)
- Conf exampls for calling with submit: [nebula-algorithm/src/main/resources/application.conf](https://github.com/vesoft-inc/nebula-algorithm/blob/master/nebula-algorithm/src/main/resources/application.conf)

### Calling with spark submit

When we call Nebula Algorithm with spark submit, on how to use perspective, it is quite similar to Exchange. [This post](https://www.siwei.io/en/nebula-livejournal/) already showed us how to do that in actions.

### Calling as a lib in code

On the other hands, we could call Nebula Algorithm in spark as a Spark Lib, the gain will be:

1) More control/customization on the output format of the algorithm
2) Possbile to perform algorithm for non-numerical vertex ID cases, see [here](https://github.com/vesoft-inc/nebula-algorithm/blob/a82d7092d928a2f3abc45a727c24afb888ff8e4f/example/src/main/scala/com/vesoft/nebula/algorithm/PageRankExample.scala#L48)



## PySpark for Nebula Graph

PySpark comes with capability to call java/scala packages inside python, thus it's also quite easy to use Spark Connector with Python.

Here I am doing this from the pyspark entrypoint in `/spark/bin/pyspark`, with connector's Jar package specified with `--driver-class-path` and `--jars`

```python
docker exec -it spark-master-0 bash
cd root
wget https://repo1.maven.org/maven2/com/vesoft/nebula-spark-connector/3.0.0/nebula-spark-connector-3.0.0.jar

/spark/bin/pyspark --driver-class-path nebula-spark-connector-3.0.0.jar --jars nebula-spark-connector-3.0.0.jar
```

Then, rather than pass `NebulaConnectionConfig` and `ReadNebulaConfig` to `spark.read.nebula`, we should instead call `spark.read.format("com.vesoft.nebula.connector.NebulaDataSource")`.

Voilà!

```python
df = spark.read.format("com.vesoft.nebula.connector.NebulaDataSource").option("type", "vertex").option("spaceName", "basketballplayer").option("label", "player").option("returnCols", "name,age").option("metaAddress", "metad0:9559").option("partitionNumber", 1).load()

>>> df.show(n=2)
+---------+--------------+---+
|_vertexId|          name|age|
+---------+--------------+---+
|player105|   Danny Green| 31|
|player109|Tiago Splitter| 34|
+---------+--------------+---+
only showing top 2 rows
```

Below are how I figured out how to make this work with almost zero scala knowledge :-P.

- [How reader should be called](https://github.com/vesoft-inc/nebula-spark-connector/blob/master/nebula-spark-connector/src/main/scala/com/vesoft/nebula/connector/package.scala)

```scala
      def loadVerticesToDF(): DataFrame = {
        assert(connectionConfig != null && readConfig != null,
               "nebula config is not set, please call nebula() before loadVerticesToDF")
        val dfReader = reader
          .format(classOf[NebulaDataSource].getName)
          .option(NebulaOptions.TYPE, DataTypeEnum.VERTEX.toString)
          .option(NebulaOptions.SPACE_NAME, readConfig.getSpace)
          .option(NebulaOptions.LABEL, readConfig.getLabel)
          .option(NebulaOptions.PARTITION_NUMBER, readConfig.getPartitionNum)
          .option(NebulaOptions.RETURN_COLS, readConfig.getReturnCols.mkString(","))
          .option(NebulaOptions.NO_COLUMN, readConfig.getNoColumn)
          .option(NebulaOptions.LIMIT, readConfig.getLimit)
          .option(NebulaOptions.META_ADDRESS, connectionConfig.getMetaAddress)
          .option(NebulaOptions.TIMEOUT, connectionConfig.getTimeout)
          .option(NebulaOptions.CONNECTION_RETRY, connectionConfig.getConnectionRetry)
          .option(NebulaOptions.EXECUTION_RETRY, connectionConfig.getExecRetry)
          .option(NebulaOptions.ENABLE_META_SSL, connectionConfig.getEnableMetaSSL)
          .option(NebulaOptions.ENABLE_STORAGE_SSL, connectionConfig.getEnableStorageSSL)
  
        if (connectionConfig.getEnableStorageSSL || connectionConfig.getEnableMetaSSL) {
          dfReader.option(NebulaOptions.SSL_SIGN_TYPE, connectionConfig.getSignType)
          SSLSignType.withName(connectionConfig.getSignType) match {
            case SSLSignType.CA =>
              dfReader.option(NebulaOptions.CA_SIGN_PARAM, connectionConfig.getCaSignParam)
            case SSLSignType.SELF =>
              dfReader.option(NebulaOptions.SELF_SIGN_PARAM, connectionConfig.getSelfSignParam)
          }
        }
  
        dfReader.load()
      }
```

  

- [How Option String should be like](https://github.com/vesoft-inc/nebula-spark-connector/blob/master/nebula-spark-connector/src/main/scala/com/vesoft/nebula/connector/NebulaOptions.scala)

```scala
# 

object NebulaOptions {

  /** nebula common config */
  val SPACE_NAME: String    = "spaceName"
  val META_ADDRESS: String  = "metaAddress"
  val GRAPH_ADDRESS: String = "graphAddress"
  val TYPE: String          = "type"
  val LABEL: String         = "label"

  /** connection config */
  val TIMEOUT: String            = "timeout"
  val CONNECTION_RETRY: String   = "connectionRetry"
  val EXECUTION_RETRY: String    = "executionRetry"
  val RATE_TIME_OUT: String      = "reteTimeOut"
  val USER_NAME: String          = "user"
  val PASSWD: String             = "passwd"
  val ENABLE_GRAPH_SSL: String   = "enableGraphSSL"
  val ENABLE_META_SSL: String    = "enableMetaSSL"
  val ENABLE_STORAGE_SSL: String = "enableStorageSSL"
  val SSL_SIGN_TYPE: String      = "sslSignType"
  val CA_SIGN_PARAM: String      = "caSignParam"
  val SELF_SIGN_PARAM: String    = "selfSignParam"

  /** read config */
  val RETURN_COLS: String      = "returnCols"
  val NO_COLUMN: String        = "noColumn"
  val PARTITION_NUMBER: String = "partitionNumber"
  val LIMIT: String            = "limit"

  /** write config */
  val RATE_LIMIT: String   = "rateLimit"
  val VID_POLICY: String   = "vidPolicy"
  val SRC_POLICY: String   = "srcPolicy"
  val DST_POLICY: String   = "dstPolicy"
  val VERTEX_FIELD         = "vertexField"
  val SRC_VERTEX_FIELD     = "srcVertexField"
  val DST_VERTEX_FIELD     = "dstVertexField"
  val RANK_FIELD           = "rankFiled"
  val BATCH: String        = "batch"
  val VID_AS_PROP: String  = "vidAsProp"
  val SRC_AS_PROP: String  = "srcAsProp"
  val DST_AS_PROP: String  = "dstAsProp"
  val RANK_AS_PROP: String = "rankAsProp"
  val WRITE_MODE: String   = "writeMode"

  val DEFAULT_TIMEOUT: Int            = 3000
  val DEFAULT_CONNECTION_TIMEOUT: Int = 3000
  val DEFAULT_CONNECTION_RETRY: Int   = 3
  val DEFAULT_EXECUTION_RETRY: Int    = 3
  val DEFAULT_USER_NAME: String       = "root"
  val DEFAULT_PASSWD: String          = "nebula"

  val DEFAULT_ENABLE_GRAPH_SSL: Boolean   = false
  val DEFAULT_ENABLE_META_SSL: Boolean    = false
  val DEFAULT_ENABLE_STORAGE_SSL: Boolean = false

  val DEFAULT_LIMIT: Int = 1000

  val DEFAULT_RATE_LIMIT: Long    = 1024L
  val DEFAULT_RATE_TIME_OUT: Long = 100
  val DEFAULT_POLICY: String      = null
  val DEFAULT_BATCH: Int          = 1000

  val DEFAULT_WRITE_MODE = WriteMode.INSERT

  val EMPTY_STRING: String = ""
}
```

- https://databricks.com/session/how-to-connect-spark-to-your-own-datasource


> feature image credit: [Sander](https://unsplash.com/photos/KABfjuSOx74)
