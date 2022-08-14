# nGQL 简明教程


> 本文旨在让新手快速了解 nGQL，掌握方向，之后可以脚踩在地上借助文档写出任何心中的 NebulaGraph 图查询。

> 本文配套视频教程还在录制中，敬请期待。

<!--more-->

## 视频

TBD

## 开始之前

本文假设你已经在文档看过[快速入门流程](https://docs.nebula-graph.com.cn/3.2.0/2.quick-start/1.quick-start-workflow/)，部署、连接过 NebulaGraph，并且看过了[常用命令](https://docs.nebula-graph.com.cn/3.2.0/2.quick-start/4.nebula-graph-crud/)。如果您还没看过这两个文档，强烈建议先快速过一遍。

### 教程目标

本教程目的在于让新手大概知道了 NebulaGraph 的查询语句后，解决“不知道什么样的查询应该用什么语句”的问题。

### nGQL 是什么

我们先强调一下概念：nGQL 是 NebulaGraph Graph Query Language 的缩写，它表示 NebulaGraph 的查询语言，而 nGQL 的语句可以不严谨地分为这几部分：

- NebulaGraph 独有 DQL 查询语句（Data Query Language）
- NebulaGraph OpenCypher DQL
- NebulaGraph DML 写语句（Data Mutation Language）
- NebulaGraph DDL Schema 语句（Data Defination Language)
- NebulaGraph Admin Queries 管理语句

这里，作为简明教程一把梭，我们只关注前两个部分，后边的内容会在 Part 2 中介绍。

### 手绘 Cheatsheet

> 大家可以报错这份单页手绘，一次了解所有 nGQL 的用法。

TBD

## NebulaGraph 独有 DQL

NebulaGraph 的独有读查询语句的设计非常简介，对初学者非常友好，结合了管道的概念，做到了只涉及了几个关键词就可以描述大多数图查询模式。

用一句话描述来说，nGQL 的独有 DQL 一共分成四类语句：

- 图拓展：`GO`
- 索引反查：`LOOKUP`
- 取属性：`FETCH PROP`
- 路径与子图：`FIND PATH` 与 `GET SUBGRAPH`

和几个特别的元素

- 管道：`|`

- 引用属性: `$` 开头的几个符号，用来描述一些特定的上下文

### 图拓展 GO

`GO` 的语义非常直观：从**给定的起点**，向外拓展，按需**返回终点**、起点的信息。

```sql
# 图拓展
GO 3 Steps FROM "player102" OVER follow YIELD dst(edge);
   ───┬───      ───┬───────      ─┬────       ──┬────── 
      │            │              │   ┌─────────┘       
      │            │              │   │                 
      │            │              │   └── 返回最后一跳边的终点
      │            │              │                     
      │            │              └────── 从 follow 这个边[出方向]探索
      │            │                                    
      │            └───────────────────── 起点是 "player102"
      │                                                 
      └────────────────────────────────── 探索 3 步
```
> 参考 [GO 语句文档](https://docs.nebula-graph.com.cn/3.2.0/3.ngql-guide/7.general-query-statements/3.go/)，了解如何：
>
> - 指定反方向拓展、双向拓展
> - 指定可变跳数拓展
> - 基于所有类型边拓展
> - 返回其他信息

### LOOKUP 基于索引反查 ID

和 `GO` 为从已知的点出发相反，`LOOKUP` 是一个类似于 SQL 里 `SELECT` 语义的关键字，它实际的作用也类似与关系型数据库中的扫表。

`LOOKUP` 需要手动明确建立相应 TAG、边类型上索引才能允许相应的查询。
#### 为什么 `LOOKUP` 需要索引？

因为 NebulaGraph 中的数据默认是按照邻接表的形式存储，在分布式设计中，扫描一个类型的点、边是非常昂贵的，所以它被默认禁止了。而创建相应的索引类似于增加了类似于表结构数据库的排序数据，可以用来做类似于 `SELECT` 的查询。

创建索引的代价是什么（增加写入负担）？索引会加速读么（不会，它是提供了 LOOKUP 的可能性，原生图的查询不需要索引加速）？等等更详细的问题请参阅我之前的[索引详解文章](https://www.siwei.io/nebula-index-explained/)。

```sql
# 索引反查
LOOKUP ON player WHERE player.name == "Tony Parker" YIELD id(vertex);
          ──┬───  ──────┬──────────────────────────  ──┬──────       
            │           │          ┌───────────────────┘             
            │           │          │                                 
            │           │          └──────────── 返回查到点的 VID
            │           │                                            
            │           └─────────────────────── 过滤条件是属性 name 的值
            │                                                        
            └─────────────────────────────────── 根据点的类别/TAG player 查询
```
> 进一步参考 [LOOKUP 语句文档](https://docs.nebula-graph.com.cn/3.2.0/3.ngql-guide/7.general-query-statements/5.lookup/)，了解如何：
>
> - 返回属性
> - 根据边的类型查询边
> - 了解 LOOKUP 查询的前提、[索引](https://docs.nebula-graph.com.cn/3.2.0/3.ngql-guide/14.native-index-statements/)，[索引详解](https://www.siwei.io/nebula-index-explained/)


### FETCH PROP 获取属性

如字面意思，如果我们知道一个点、边的 ID，想要获取它上边的属性，这时候我们要用 `FETCH PROP` 而非 `LOOKUP`。

```sql
# 取属性
FETCH PROP ON player "player100" YIELD properties(vertex);
              ──┬───  ────┬─────       ─────────┬──────── 
                │         │         ┌───────────┘         
                │         │         │                     
                │         │         └─────── 返回点的 player TAG 下所有属性
                │         │                               
                │         └───────────────── 从 "player100" 这个点获取
                │                                         
                └─────────────────────────── 获取 player 这个 TAG 下的属性
```
> 进一步参考 [FETCH PROP 语句文档](https://docs.nebula-graph.com.cn/3.2.0/3.ngql-guide/7.general-query-statements/4.fetch/)，了解如何：
>
> - 返回某一个属性
> - 获取给定边的属性

### 路径查找 FIND PATH

如果我们从给定的起点、终点中，找到之间的所有路径，一定要用 `FIND PATH`

```sql
# 起点终点间路径
FIND SHORTEST PATH FROM "player102" TO "team204" OVER * \   
     ──┬─────            ───────────┬─────────── ───┬───    
  YIELD│path AS p; ┌────────────────┘               │       
       │────┬────  │     ┌──────────────────────────┘       
       │    │      │     │                                  
       │    │      │     └───────── 经由所有类型的边出向探索
       │    │      │                                        
       │    │      └─────────────── 从给定的起点、终点 VID
       │    │                                               
       │    └────────────────────── 返回路径为 p 列
       │                                                    
       └─────────────────────────── 查找最短路径
```
> 进一步参考 [FIND PATH 语句文档](https://docs.nebula-graph.com.cn/3.2.0/3.ngql-guide/16.subgraph-and-path/2.find-path/)，了解如何：
>
> - 返回路径中的属性
> - 设定拓展方向



### 单点子图 GET SUBGRAPH

和路径查找类似，但是我们只给定一个起点和拓展部署，用 `GET SUBGRAPH` 可以帮我们获取同样的 BFS 出去的子图

```sql
# 单点 BFS 子图
GET SUBGRAPH 5 STEPS FROM "player101" \           
             ───┬─── ─────┬──────────             
  YIELD VERTICES AS nodes, EDGES AS relationships;
        ────┬───┼─────────┼───────────────────────
   ┌────────┘                                     
   │            │         └─────── 从 "player101" 开始触发
   │                                              
   │            └───────────────── 获取 5 步的探索
   │                                              
   └────────────────────────────── 返回所有的点、边
```
> 进一步参考 [GET SUBGRAPH 语句文档](https://docs.nebula-graph.com.cn/3.2.0/3.ngql-guide/16.subgraph-and-path/1.get-subgraph/)，了解如何：
>
> - 返回带有属性的点、边
> - 设定拓展方向

### 利用管道和属性引用符
NebulaGraph 的管道设计和 Unix-Shell 的设计很像，可以将简单的几种语句结合起来，有强大的表达力。

```sql
# 使用通道
GO FROM "player100" OVER follow YIELD dst(edge) AS did  | \  
    ─────┬──────────────────────────────────────────── ─┬─   
  GO FROM│$-.did OVER follow YIELD dst(edge);           │    
         │────┬──     ┌─────────────────────────────────┘    
         │    │       │                                      
         │    │       └──────── 管道将左边的 AS 输出作为右边语句输入
         │    │                                              
         │    └──────────────── 从管道左边的 did 属性开始探索
         │
         └───────────────────── 第一个查询语句
```

> 进一步参考 [引用属性文档](https://docs.nebula-graph.com.cn/3.2.0/3.ngql-guide/5.operators/5.property-reference/)、[管道文档](https://docs.nebula-graph.com.cn/3.2.0/3.ngql-guide/5.operators/4.pipe/)了解：
>
> - 更多属性引用定义
> - 更多例子
> - 结合 LOOKUP, GO, FETCH 的语句

除了以上的集中表达之外，NebulaGraph 独有查询语句还有聚合的表达参考 [GROUP-BY](https://docs.nebula-graph.com.cn/3.2.0/3.ngql-guide/8.clauses-and-options/group-by/)，另外在文档里还有一个 [Cheatsheet](https://docs.nebula-graph.com.cn/3.2.0/2.quick-start/6.cheatsheet-for-ngql-command/) 供大家查询一些复杂一点查询的例子。

### NebulaGraph OpenCypher DQL

从 2.0 起，OpenCypher 的 `MATCH` 语句也被 NebulaGraph 原生支持了，虽然这里是一个方言（有一些细节差异）。

```cypher
MATCH <pattern> [<clause_1>]  RETURN <output>  [<clause_2>];
```

MATCH 的基本表达是以 `(v:tag_a)` 包裹的点 `-->` 或者 `<-[:edge_type_1]-` 表达的边组成的模式，与 `RETURN` 表达的输出。

如果您从 Cypher 的查询语句入门图数据库，可以从下边几个例子了解到几个 NebulaGraph 里的细节差异：

- 增加了 `WHERE id(v) == "foo"` 的表达
- `==` 表达相等判断而不是 `=`
- 点的属性表达需要填写 TAG，例如 `v3.player.name` 而不是 `v3.name`

```cypher
MATCH (v:player{name:"Tim Duncan"})-->(v2)<--(v3) \
    RETURN v3.player.name AS Name;

MATCH (v:player) \
    WHERE not (v)--() \
    RETURN v;

MATCH (v:player)--(v2) \
    WHERE id(v2) IN ["player101", "player102"] \
    RETURN v;

MATCH (m)-[]->(n) WHERE id(m)=="player100" \
OPTIONAL MATCH (n)-[]->(l) WHERE id(n)=="player125" \
    RETURN id(m),id(n),id(l);
```

> 进一步参考 [MATCH 文档](https://docs.nebula-graph.com.cn/3.2.0/3.ngql-guide/7.general-query-statements/2.match/)了解：

- 更多例子
- 可变跳数的 MATCH 表达
- 多 MATCH
- OPTIONAL MATCH

> 题图版权：DALL·E Open-AI，[原图](./featured-image-raw.webp)
>
> Featured image was generated with key words: learning spells of the nebula magic, with DALL·E Open-AI.


