# Nebula Java Happy Parsing Guide


> How to parse nebula graph data in an interactive way and what are the best practices?
>
> I will show you an easier way in this article 😁.

<!--more-->

> updated: 2022-Aug-10, adapted to nebulagraph 3.x

## Prepare for the Java REPL

Thanks to https://github.com/albertlatacz/java-repl/ we could play with/debug this in an interactive way, and all we need is to leverage its docker image to have all the envrioment in a clean and quick way:

```bash
docker pull albertlatacz/java-repl
docker run --rm -it \
    --network=nebula-net \
    -v ~:/root \
    albertlatacz/java-repl \
    bash
apt update -y && apt install ca-certificates -y
wget https://dlcdn.apache.org/maven/maven-3/3.8.6/binaries/apache-maven-3.8.6-bin.tar.gz --no-check-certificate

tar xzvf apache-maven-3.8.6-bin.tar.gz

wget https://github.com/vesoft-inc/nebula-java/archive/refs/tags/v3.0.0.tar.gz
tar xzvf v3.0.0.tar.gz
cd nebula-java-3.0.0/
../apache-maven-3.8.6/bin/mvn dependency:copy-dependencies
../apache-maven-3.8.6/bin/mvn -B package -Dmaven.test.skip=true

java -jar ../javarepl/javarepl.jar
```

Now, after executing `java -jar ../javarepl/javarepl.jar` we are in a Java Shell(REPL), this enable us to execute Java code in an interactive way without wasting time and patience in the slow path(code --> build --> execute --> add print --> build), isn't that cool?

Like this:
```java
root@a2e26ba62bb6:/javarepl/nebula-java-3.0.0# java -jar ../javarepl/javarepl.jar

Welcome to JavaREPL version 428 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_111)
Type expression to evaluate, :help for more options or press tab to auto-complete.
Connected to local instance at http://localhost:43707

java> System.out.println("Hello, World!");
Hello, World!
java>
```

Now we are in the java REPL, let's introduce all the class path needed and do the imports in one go:

```java
:cp /javarepl/nebula-java-3.0.0/client/target/client-3.0.0.jar
:cp /javarepl/nebula-java-3.0.0/client/target/dependency/fastjson-1.2.78.jar
:cp /javarepl/nebula-java-3.0.0/client/target/dependency/slf4j-api-1.7.25.jar
:cp /javarepl/nebula-java-3.0.0/client/target/dependency/slf4j-log4j12-1.7.25.jar
:cp /javarepl/nebula-java-3.0.0/client/target/dependency/commons-pool2-2.2.jar
:cp /javarepl/nebula-java-3.0.0/client/target/dependency/log4j-1.2.17.jar

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

And let's connect it to the nebula graph, please replace your graphD IP and Port here, and execute them under the propmt string of `java>`:

```java
NebulaPoolConfig nebulaPoolConfig = new NebulaPoolConfig();
nebulaPoolConfig.setMaxConnSize(10);
List<HostAddress> addresses = Arrays.asList(new HostAddress("192.168.8.127", 9669));
NebulaPool pool = new NebulaPool();
pool.init(addresses, nebulaPoolConfig);
Session session = pool.getSession("root", "nebula", false);
```

## The `execute` for ResultSet

First let's check what we can do with a simple query:

```java
ResultSet resp = session.execute("USE basketballplayer;MATCH (n:player) WHERE n.name==\"Tim Duncan\" RETURN n");
```

Now you could play with it:
> Reference: [client/graph/data/ResultSet.java](https://github.dev/vesoft-inc/nebula-java/blob/master/client/src/main/java/com/vesoft/nebula/client/graph/data/ResultSet.java)


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

Remember our item is actually a vertex?

```ngql
(root@nebula) [basketballplayer]> match (n:player) WHERE n.name == "Tim Duncan" return n
+----------------------------------------------------+
| n                                                  |
+----------------------------------------------------+
| ("player100" :player{age: 42, name: "Tim Duncan"}) |
+----------------------------------------------------+
Got 1 rows (time spent 2116/44373 us)
```

Let's see what(methods) can be done towards a value?

```java
v = Class.forName("com.vesoft.nebula.Value")
v.getDeclaredMethods()
```
We could tell it's quite Primitive on what `com.vesoft.nebula.Value` provided, thus we should use the ValueWrapper(or use executeJson actually) instead.

To get a row of the result via iteration(as its a java iterable), we just follow how the example looped the result:

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

As shown in above, the result value/item could be in properties string/int etc... or in graph semantic vertex, edge, path, we should use correspond `asXxxx` methods:

```java
java> v = wrappedValueList.get(0)
com.vesoft.nebula.client.graph.data.ValueWrapper v = ("player100" :player {name: "Tim Duncan", age: 42})
java> v.asNode()
com.vesoft.nebula.client.graph.data.Node res16 = ("player100" :player {name: "Tim Duncan", age: 42})
java> node = v.asNode()
com.vesoft.nebula.client.graph.data.Node node = ("player100" :player {name: "Tim Duncan", age: 42})
```

Btw, it's also possible to play with it with reflections(we imported already):
> Of courese we could also check via [client/graph/data/ResultSet.java](https://github.dev/vesoft-inc/nebula-java/blob/master/client/src/main/java/com/vesoft/nebula/client/graph/data/ResultSet.java)

```java
java> rClass=Class.forName("com.vesoft.nebula.client.graph.data.ResultSet")
java.lang.Class r = class com.vesoft.nebula.client.graph.data.ResultSet
java> rClass.getDeclaredMethods()
java.lang.reflect.Method[] res20 = [public java.util.List com.vesoft.nebula.client.graph.data.ResultSet.getColumnNames(), public int com.vesoft.nebula.client.graph.data.ResultSet.rowsSize(), public com.vesoft.nebula.client.graph.data.ResultSet$Record com.vesoft.nebula.client.graph.data.ResultSet.rowValues(int), public java.util.List com.vesoft.nebula.client.graph.data.ResultSet.colValues(java.lang.String), public java.lang.String com.vesoft.nebula.client.graph.data.ResultSet.getErrorMessage(), public boolean com.vesoft.nebula.client.graph.data.ResultSet.isSucceeded(), public int com.vesoft.nebula.client.graph.data.ResultSet.getErrorCode(), public java.lang.String com.vesoft.nebula.client.graph.data.ResultSet.getSpaceName(), public int com.vesoft.nebula.client.graph.data.ResultSet.getLatency(), public com.vesoft.nebula.graph.PlanDescription com.vesoft.nebula.client.graph.data.ResultSet.getPlanDesc(), public java.util.List com.vesoft.nebula.client.graph.data.ResultSet.getRows(), public java.lang.String com.vesoft.nebula.client.graph.data.ResultSet.getComment(), public java.lang.String com.vesoft.nebula.client.graph.data.ResultSet.toString(), public boolean com.vesoft.nebula.client.graph.data.ResultSet.isEmpty(), public java.util.List com.vesoft.nebula.client.graph.data.ResultSet.keys()]
```


```java
java> nodeClass=Class.forName("com.vesoft.nebula.client.graph.data.Node")
java.lang.Class nodeClass = class com.vesoft.nebula.client.graph.data.Node
java> nodeClass.getDeclaredMethods()
java.lang.reflect.Method[] res20 = [public boolean com.vesoft.nebula.client.graph.data.Node.hasTagName(java.lang.String), public boolean com.vesoft.nebula.client.graph.data.Node.hasLabel(java.lang.String), public java.util.List com.vesoft.nebula.client.graph.data.Node.tagNames(), public java.util.HashMap com.vesoft.nebula.client.graph.data.Node.properties(java.lang.String) throws java.io.UnsupportedEncodingException, public java.util.List com.vesoft.nebula.client.graph.data.Node.labels(), public boolean com.vesoft.nebula.client.graph.data.Node.equals(java.lang.Object), public java.lang.String com.vesoft.nebula.client.graph.data.Node.toString(), public java.util.List com.vesoft.nebula.client.graph.data.Node.values(java.lang.String), public int com.vesoft.nebula.client.graph.data.Node.hashCode(), public com.vesoft.nebula.client.graph.data.ValueWrapper com.vesoft.nebula.client.graph.data.Node.getId(), public java.util.List com.vesoft.nebula.client.graph.data.Node.keys(java.lang.String) throws java.io.UnsupportedEncodingException]
```
It comes with methods like `tagNames()`, `properties()`, `labels()` etc...

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

We only demestrated the `RETURN n` case which is a vertex, but for sure you could explore other types like: edge, path or datetime, string etc., right?


## The `executeJson` method

Since 2.6, nebule finally supports json string response and we could do this:

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
java>
```

And I believe you know much better than I do with it.

## Conclusion

- If we go with JSON response, it'll be easier for you to have everything parsed.
- If we have to deal with resultSet object, just use the ValueWrapper asNode() if the value is a vertex, use asRelationship if value is an edge and asPath() if the value is a path.
    - With the REPL tool shown together with java reflection and source code, it's possbile to inspect on how data could be parsed.

Happy Graphing!



> Picture Credit：[leunesmedia](https://unsplash.com/photos/0wIHsm2_1fc) 

