# Nebula Graph 的 Java 数据解析实践与指导


> 如何快速、即时、符合直觉地去处理 Nebula Java Client 中的数据解析？读这一篇就够了。

<!--more-->

## 关键步骤：几行准备一个干净的交互式 Nebula Java REPL 环境

多亏了 [Java-REPL](https://github.com/albertlatacz/java-repl/) 我们可以很方便地（像 iPython 那样）去实时交互地调试、分析 Nebula Java 客户端，我们用它的 Docker 镜像可以很干净的去搞定：

```bash
docker pull albertlatacz/java-repl
docker run --rm -it \
    --network=nebula-docker-compose_nebula-net \
    -v ~:/root \
    albertlatacz/java-repl \
    bash
apt update -y && apt install ca-certificates -y
wget https://dlcdn.apache.org/maven/maven-3/3.8.4/binaries/apache-maven-3.8.4-bin.tar.gz --no-check-certificate

tar xzvf apache-maven-3.8.4-bin.tar.gz

wget https://github.com/vesoft-inc/nebula-java/archive/refs/tags/v2.6.1.tar.gz
tar xzvf v2.6.1.tar.gz
cd nebula-java-2.6.1/
../apache-maven-3.8.4/bin/mvn dependency:copy-dependencies
../apache-maven-3.8.4/bin/mvn -B package -Dmaven.test.skip=true

java -jar ../javarepl/javarepl.jar
```
这时候，在执行完 `java -jar ../javarepl/javarepl.jar` 之后，我们就进入了交互式的 Java Shell（REPL），我们可以无需做编译，执行，print 这样的慢反馈来调试和研究我们的代码了，是不是很方便？

```java
root@a2e26ba62bb6:/javarepl/nebula-java-2.6.1# java -jar ../javarepl/javarepl.jar

Welcome to JavaREPL version 428 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_111)
Type expression to evaluate, :help for more options or press tab to auto-complete.
Connected to local instance at http://localhost:43707

java> System.out.println("Hello, World!");
Hello, World!
java>
```
首先我们在 `java>` 提示符下，这些来把必须的类路径和导入：

```java
:cp /javarepl/nebula-java-2.6.1/client/target/client-2.6.1.jar
:cp /javarepl/nebula-java-2.6.1/client/target/dependency/fastjson-1.2.78.jar
:cp /javarepl/nebula-java-2.6.1/client/target/dependency/slf4j-api-1.7.25.jar
:cp /javarepl/nebula-java-2.6.1/client/target/dependency/slf4j-log4j12-1.7.25.jar
:cp /javarepl/nebula-java-2.6.1/client/target/dependency/commons-pool2-2.2.jar
:cp /javarepl/nebula-java-2.6.1/client/target/dependency/log4j-1.2.17.jar

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import com.vesoft.nebula.ErrorCode;
import com.vesoft.nebula.client.graph.NebulaPoolConfig;
import com.vesoft.nebula.client.graph.data.CASignedSSLParam;
import com.vesoft.nebula.client.graph.data.HostAddress;
import com.vesoft.nebula.client.graph.data.ResultSet;
import com.vesoft.nebula.client.graph.data.SelfSignedSSLParam;
import com.vesoft.nebula.client.graph.data.ValueWrapper;
import com.vesoft.nebula.client.graph.net.NebulaPool;
import com.vesoft.nebula.client.graph.net.Session;
import java.io.UnsupportedEncodingException;
import java.util.Arrays;
import java.util.List;
import java.util.concurrent.TimeUnit;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.lang.reflect.*;
```
我们可以从这 Java 环境连接到 Nebula Graph 里，下边的例子里我用了自己的 GraphD 的 IP 和端口作为例子：

```java
NebulaPoolConfig nebulaPoolConfig = new NebulaPoolConfig();
nebulaPoolConfig.setMaxConnSize(10);
List<HostAddress> addresses = Arrays.asList(new HostAddress("192.168.8.127", 9669));
NebulaPool pool = new NebulaPool();
pool.init(addresses, nebulaPoolConfig);
Session session = pool.getSession("root", "nebula", false);
```

## 通过调用 `execute` 方法获得不太容易懂的 ResultSet 对象

刚接触这里的大家一定对这个 ResultSet 对象有些愁，借助我们的环境，咱们来十分钟把它搞通吧，这里我们执行一个简单的返回 Vertex 顶点的结果看看：

```java
ResultSet resp = session.execute("USE basketballplayer;MATCH (n:player) WHERE n.name==\"Tim Duncan\" RETURN n");
```

这里我们可以参考 ResultSet 的代码：

> Reference: [client/graph/data/ResultSet.java](https://github.dev/vesoft-inc/nebula-java/blob/master/client/src/main/java/com/vesoft/nebula/client/graph/data/ResultSet.java)

好吧，其实可以先不看，跟着我的教程往下走吧，我们知道结果都是二维的表，ResultSet 提供了常见的针对行、列的一些方法，通常，我们是获取每一行，然后解析它，而关键的问题是每一个值要怎么处理，对吧。

```java
java> resp.isSucceeded()
java.lang.Boolean res9 = true

java> resp.rowsSize()
java.lang.Integer res16 = 1

java> rows = resp.getRows()
java.util.ArrayList rows = [Row (
  values : [
    <Value vVal:Vertex (
        vid : <Value sVal:70 6c 61 79 65 72 31 30 30>,
        tags : [
          Tag (
              name : 70 6C 61 79 65 72,
              props : {
                [B@5264a468 : <Value iVal:42>
                [B@496b8e10 : <Value sVal:54 69 6d 20 44 75 6e 63 61 6e>
              }
            )
        ]
      )>
  ]
)]
    
java> row0 = resp.rowValues(0)
java.lang.Iterable<com.vesoft.nebula.client.graph.data.ValueWrapper> res10 = ColumnName: [n], Values: [("player100" :player {name: "Tim Duncan", age: 42})]

```

我们其实回到这次的 query ，其实是返回一个 vertex：顶点：

```ngql
(root@nebula) [basketballplayer]> match (n:player) WHERE n.name == "Tim Duncan" return n
+----------------------------------------------------+
| n                                                  |
+----------------------------------------------------+
| ("player100" :player{age: 42, name: "Tim Duncan"}) |
+----------------------------------------------------+
Got 1 rows (time spent 2116/44373 us)
```

通过上边的几个方法，我们其实能够获得这个顶点的值：

```java
v = Class.forName("com.vesoft.nebula.Value")
v.getDeclaredMethods()
```
然而，我们可以看出来这个 `com.vesoft.nebula.Value` 的值的类提供的方法特别特别原始，这也是让大家犯愁的原因，在这个教程里最重要的一个带走的经验（除了利用 REPL之外）就是：除非必要，不要去取这个原始的类，我们应该去取得 `ValueWrapper` 封装之后的值！！！

> 注意：其实我们有更轻松地方法，就是用 executeJson 直接获得 JSON string，别担心，会在后边提到，不过这个方法要 2.6 之后才支持。

那么问题来了，如何使用 `ValueWrapper` 封装呢？其实答案已经在上边了，大家可以回去看看，resp.rowValues(0) 的类型正是 ValueWrapper 的可迭代对象！

所以，正确打开方式是迭它！迭它！迭它！其实这个就是代码库里的 GraphClientExample 的一部分例子了，我们把它迭代取出来，放到 `wrappedValueList` 里慢慢把玩：

```java
import java.util.ArrayList;
import java.util.List;
List<ValueWrapper> wrappedValueList = new ArrayList<>();

for (int i = 0; i < resp.rowsSize(); i++) {
    ResultSet.Record record = resp.rowValues(i);
    for (ValueWrapper value : record.values()) {
        wrappedValueList.add(value);
        if (value.isLong()) {
            System.out.printf("%15s |", value.asLong());
        }
        if (value.isBoolean()) {
            System.out.printf("%15s |", value.asBoolean());
        }
        if (value.isDouble()) {
            System.out.printf("%15s |", value.asDouble());
        }
        if (value.isString()) {
            System.out.printf("%15s |", value.asString());
        }
        if (value.isTime()) {
            System.out.printf("%15s |", value.asTime());
        }
        if (value.isDate()) {
            System.out.printf("%15s |", value.asDate());
        }
        if (value.isDateTime()) {
            System.out.printf("%15s |", value.asDateTime());
        }
        if (value.isVertex()) {
            System.out.printf("%15s |", value.asNode());
        }
        if (value.isEdge()) {
            System.out.printf("%15s |", value.asRelationship());
        }
        if (value.isPath()) {
            System.out.printf("%15s |", value.asPath());
        }
        if (value.isList()) {
            System.out.printf("%15s |", value.asList());
        }
        if (value.isSet()) {
            System.out.printf("%15s |", value.asSet());
        }
        if (value.isMap()) {
            System.out.printf("%15s |", value.asMap());
        }
    }
    System.out.println();
}
```

上边这些很丑的 if 就是关键了，我们知道 query 的返回值可能是多种类型的，他们分为：

- 图语义的：点、边、路径
- 数据类型：String，日期，列表，集合 等等等

这里的关键是，我们要使用 `ValueWrapper` 为我们准备好的 `asXxx` 方法，如果这个值是一个顶点，么这个 Xxx 就是 Node，同理如果是边的话，这个 Xxx 就是 Relationship。

所以，我给大家看看咱们这个返回点结果的情况下的 `asNode()` 方法：

```java
java> v = wrappedValueList.get(0)
com.vesoft.nebula.client.graph.data.ValueWrapper v = ("player100" :player {name: "Tim Duncan", age: 42})
java> v.asNode()
com.vesoft.nebula.client.graph.data.Node res16 = ("player100" :player {name: "Tim Duncan", age: 42})
java> node = v.asNode()
com.vesoft.nebula.client.graph.data.Node node = ("player100" :player {name: "Tim Duncan", age: 42})
```

顺便说一下，借助于 Java 的 reflection ，我们可以在这个交互程序里做类似于 Python 里 `dir()` 的事情，实时地去获取一个类支持的方法，像这样，省去了查代码。

```java
java> rClass=Class.forName("com.vesoft.nebula.client.graph.data.ResultSet")
java.lang.Class r = class com.vesoft.nebula.client.graph.data.ResultSet
java> rClass.getDeclaredMethods()
java.lang.reflect.Method[] res20 = [public java.util.List com.vesoft.nebula.client.graph.data.ResultSet.getColumnNames(), public int com.vesoft.nebula.client.graph.data.ResultSet.rowsSize(), public com.vesoft.nebula.client.graph.data.ResultSet$Record com.vesoft.nebula.client.graph.data.ResultSet.rowValues(int), public java.util.List com.vesoft.nebula.client.graph.data.ResultSet.colValues(java.lang.String), public java.lang.String com.vesoft.nebula.client.graph.data.ResultSet.getErrorMessage(), public boolean com.vesoft.nebula.client.graph.data.ResultSet.isSucceeded(), public int com.vesoft.nebula.client.graph.data.ResultSet.getErrorCode(), public java.lang.String com.vesoft.nebula.client.graph.data.ResultSet.getSpaceName(), public int com.vesoft.nebula.client.graph.data.ResultSet.getLatency(), public com.vesoft.nebula.graph.PlanDescription com.vesoft.nebula.client.graph.data.ResultSet.getPlanDesc(), public java.util.List com.vesoft.nebula.client.graph.data.ResultSet.getRows(), public java.lang.String com.vesoft.nebula.client.graph.data.ResultSet.getComment(), public java.lang.String com.vesoft.nebula.client.graph.data.ResultSet.toString(), public boolean com.vesoft.nebula.client.graph.data.ResultSet.isEmpty(), public java.util.List com.vesoft.nebula.client.graph.data.ResultSet.keys()]
```
这样：

```java
java> nodeClass=Class.forName("com.vesoft.nebula.client.graph.data.Node")
java.lang.Class nodeClass = class com.vesoft.nebula.client.graph.data.Node
java> nodeClass.getDeclaredMethods()
java.lang.reflect.Method[] res20 = [public boolean com.vesoft.nebula.client.graph.data.Node.hasTagName(java.lang.String), public boolean com.vesoft.nebula.client.graph.data.Node.hasLabel(java.lang.String), public java.util.List com.vesoft.nebula.client.graph.data.Node.tagNames(), public java.util.HashMap com.vesoft.nebula.client.graph.data.Node.properties(java.lang.String) throws java.io.UnsupportedEncodingException, public java.util.List com.vesoft.nebula.client.graph.data.Node.labels(), public boolean com.vesoft.nebula.client.graph.data.Node.equals(java.lang.Object), public java.lang.String com.vesoft.nebula.client.graph.data.Node.toString(), public java.util.List com.vesoft.nebula.client.graph.data.Node.values(java.lang.String), public int com.vesoft.nebula.client.graph.data.Node.hashCode(), public com.vesoft.nebula.client.graph.data.ValueWrapper com.vesoft.nebula.client.graph.data.Node.getId(), public java.util.List com.vesoft.nebula.client.graph.data.Node.keys(java.lang.String) throws java.io.UnsupportedEncodingException]
```

看到这里，大家应该看到了这个封装了的 ValueWrapper 的好处了吧？它提供了方便的符合直觉的方法，对于 Node 类型来说，它提供了 `tagNames()`, `properties()`, `labels()` 等等非常好用的方法：

```java
java> node.properties("player")
java.util.HashMap res11 = {name="Tim Duncan", age=42}
java> node.tagNames()
java.util.ArrayList res12 = [player]
java> node.labels()
java.util.ArrayList res13 = [player]
java> node.values("player")
java.util.ArrayList res14 = [42, "Tim Duncan"]
```

我们这里只展示了顶点数据类型的处理、解析方式（`RETURN n`），像其他的数据类型比如边（edge）路径（path）或者地理数据、时间数据，用这种方式（看有什么方法，再交互地去试试方法怎么用）也是一样的，对吧？


## 直接返回 JSON 的 `executeJson` 方法

最后，好消息是，从 2.6 开始，nebula 可以直接返回 JSON 的 String 了，我们上边的纠结也都不是必要的了：

```java
java> String resp_json = session.executeJson("USE basketballplayer;MATCH (n:player) WHERE n.name==\"Tim Duncan\" RETURN n");

java.lang.String resp_json = "
{
   "errors":[
      {
         "code":0
      }
   ],
   "results":[
      {
         "spaceName":"basketballplayer",
         "data":[
            {
               "meta":[
                  {
                     "type":"vertex",
                     "id":"player100"
                  }
               ],
               "row":[
                  {
                     "player.age":42,
                     "player.name":"Tim Duncan"
                  }
               ]
            }
         ],
         "columns":[
            "n"
         ],
         "errors":{
            "code":0
         },
         "latencyInUs":4761
      }
   ]
}
"
```
我相信大家肯定比我更擅长处理 JSON 的结果了哈~~

## 结论

- 如果我们有条件（2.6以后）用 JSON，情况会很容易，可能大家不太需要本文的方法（不过有交互环境还是很方便吧？）
- 如果我们不得不和 resultSet 打交道，记得用 ValueWrapper ，因为我们可以用 asNode()，asRelationship() 和 asPath() ，封装之后的值比原始的值可爱太多了！
    - 通过 REPL 工具，结合 Java 的 reflection 加上 源代码本身，分析数据的处理将变得异常顺滑

Happy Graphing!

> Picture Credit：[leunesmedia](https://unsplash.com/photos/0wIHsm2_1fc) 

