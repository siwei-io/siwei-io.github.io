# Nebula Index Explained


> Nebula Graph Native Index explained, why `index not found`? When should I use Nebula Index and full-text index?

The term of Nebula Graph Index is quite similar to index in RDBMS, while, they are not the same. It's noticed when getting started with Nebula Graph, the index confused some of the users in first glance on

- What exactly Nebula Graph Index is.
- When I should use it.
- How it impacts the performance.

Today I'm gonna walk you through the index in Nebula Graph.

Let's get started!

## What exactly Nebula Graph Index is

TL;DR, Nebula Graph Index is only to be used to enable pure-prop-condition queries

- Not for graph walking through edges.
- It's an prerequisite for such query.

### pure-prop-condition queries

We know that in RDBMS, an INDEX is to create a duplicated sorted DATA to enable QUERY with condition filtering on the sorted data, to **accelerate the query in read** and involves extra writes during the write.

> Note: in RDBMS/Tabular DB, an INDEX on some columns means to create extra data that are sorted on those columns to make query with those columns' condition to be scanned faster, rather than scanning from the original table data sorted based on the key only.

In Nebula Graph, the INDEX is to create a duplicated sorted **Vertex/Edge PROP DATA** to **enable the part of QUERY**(it's a must rather than accelerate it) like The following to GET data from PROP Conditions: or I call it the pure-prop-condition queries:

```sql
#### Queries relying on Nebula Graph Index

# query 0 pure-prop-condition query
LOOKUP ON tag1 WHERE col1 > 1 AND col2 == "foo" \
    YIELD tag1.col1 as col1, tag1.col3 as col3;

# query 1 pure-prop-condition query
MATCH (v:player { name: 'Tim Duncan' })-->(v2:player) \
        RETURN v2.player.name AS Name;
```

The pure-prop-condition queries, like above nGQL lines are literally to "Find VID/EDGE only based on given the propertiy condtions". On the contrary, the following, are not pure-prop-condition queries:

```sql
#### Queries not based on Nebula Graph Index

# query 2, walk query starting from given vertex VID: "player100"

GO FROM "player100" OVER follow REVERSELY \
        YIELD src(edge) AS id | \
    GO FROM $-.id OVER serve \
        WHERE properties($^).age > 20 \
        YIELD properties($^).name AS FriendOf, properties($$).name AS Team;

# query 3, walk query starting from given vertex VID: "player101" or "player102"

MATCH (v:player { name: 'Tim Duncan' })--(v2) \
        WHERE id(v2) IN ["player101", "player102"] \
        RETURN v2.player.name AS Name;
```

If we look into `query 1` and `query 3`, where condition on vertex on tag:player are both `{ name: 'Tim Duncan' }` though:

- For `query 3` , the index is not required as the query will be started from known vertex ID in `["player101", "player102"]` and thus:
  - It'll directly fetch vertex Data from `v2`'s vertex IDs
  - then to GetNeighbors(): walk through edges of `v2`, GetVertices() for next hop: `v` and filter based on property: `name`
- For `query 1` , the query has to start from `v` due to no known vertex IDs were provided:
  - It'll do IndexScan() first to find all vertices only with property condtion of `{ name: 'Tim Duncan' }`
  - Then, GetNeighbors(): walk through edges of `v`, GetVertices() for next hop: `v2` 

Now, we could know the whole point is on **whether to know the vertexID(s)**. You could check their execution plan with PROFILE or EXPLAIN like the follow:

| `query 1`, requires/based on index(on tag: player), pure prop condition query | `query 3`, no index required, query starting from known vertex IDs |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ![query-based-on-index](./query-based-on-index.webp)         | ![query-requires-no-index](./query-requires-no-index.webp)   |

### Why Nebula Graph index is a must in pure-prop-condition queries

It's because Nebula Graph stores data in a distributed and graph-oriented way, the full scan of data was condiser too expensive to be allowed(`index not found` will occur when it's not created in pure-prop-condition queries).

> Note: from v3.0, it's possible to do TopN Scan without INDEX, where the `LIMIT <n>` is used, this is different from the fullscan case(INDEX is a must), which will be explained later.
>
> ```cypher
> MATCH (v:player { name: 'Tim Duncan' })-->(v2:player) \
>         RETURN v2.player.name AS Name LIMIT 3;
> ```

### Why pure-prop-condition queries only(requiring index)

**graph-queries** vs **pure-prop-condition queries**

- graph-queries, see `query 2` and `query 3`, are to walk through edges all the way for the given vertices
- pure-prop-condition queries, see `query 0` and `query 1`, are to find vertex only on given prop condtions

In Nebula Graph, the data is structured in a way to enable fast graph-queries, and is already indexed/sorted on vertex ID(for both vertex and edge) in raw data, where GetNeighbors() of given vertex is cheap and fast due to the locality/stored continuously(pysically linked).



So in summary:

> Nebula Graph Index is sorted prop data to find vertex or edge on given pure prop conditions.

## Facts on Nebula Graph Index

To understand more details/limitations/cost of Nebula, let's reveal more on its design and here are some facts:

- Index Data is stored and sharded together with Vertex Data

- Only Left Match: It's RocksDB Prefix Scan under the hood

- Cost:

  - Write Path: Extra Data + Extra Read
  - Read Path: RBO, Fan Out, TopN Push Down

- Data Full Scan TopN Sample(not fullscan) is supported w/o Index
  - `LOOKUP ON t YIELD t.name | LIMIT 1`

  - ```cypher
    MATCH (v:player { name: 'Tim Duncan' })-->(v2:player) \
            RETURN v2.player.name AS Name LIMIT 3;
    ```

The key info can be seen from one of my [sketch notes](https://www.siwei.io/en/sketch-notes/):

![](https://www.siwei.io/sketches/nebula-index-demystified/nebula-index-demystified-en.webp)

> We should notice that only the left match is supported in pure-prop-condition queries. For queries like wildcard or reguler-expression, Full-text Index/Search is to be used, which leveraged an external elastic search integrated with nebula: please check [Nebula Graph Full text index](https://docs.nebula-graph.io/3.0.0/4.deployment-and-installation/6.deploy-text-based-index/2.deploy-es/) for more.

With this sketch note, we could see

- Local Index Design
  - The index is stored and shared locally together with the graph data.
  - It's sorting based on prop value, and the matching is underlying a rocksDB prefix scan, that's why only left match is supported()

- Write path
  - The index enables the RDBMS-like Prop Condition Query with cost in the write path including not only the extra write, but also, random read, to ensure the data consistency.
  - Index Data write is done in a sync way

- Read path
  - In pure-prop-condition queries, in GraphD, the index will be selected with Rule-based-optimization like this example, where, in a rule, the col2 to be sorted first is considered optimal with the condition: col2 equals 'foo'.
  - After the index was chosen, index-scan request will be fanout to storageD instances, and in the case of filters like LIMIT N, it will be pushed down to the storage side to reduce data payload.
    - Note: not shown in the sketch but actually from v3.0, the nebula graph allows LIMIT N Sample Prop condition query like this w/o index, which is underlying pushing down the LIMIT filter to storage side.


Take aways:

- Use index only when we have to, as it's costly in write cases and if limit N sample is allowed and fast enough, we can use that instead, if not: use index.
- Index is left match
  - composite index order matters, should be created carefully.
  - for full-text search use case, use [full-text index](https://docs.nebula-graph.io/3.0.0/4.deployment-and-installation/6.deploy-text-based-index/2.deploy-es/) instead.



## How to use the index

We should always refer to the [documentation](https://docs.nebula-graph.io/3.0.0/3.ngql-guide/14.native-index-statements/), and I just put some highlights on this here:

- To create an index on a tag or edge type to specify a list of props in the order that we need.

  - `CREATE INDEX`

- If an index was created after existing data was inserted, we need to trigger an index async rebuild job, as the index data will be written in sync way only when index is created.

  - `REBUILD INDEX`

- We can see the index status after `REBUILD INDEX` issued.

  - `SHOW INDEX STATUS`

- Queries levering index could be LOOKUP, and with the pipeline, in most cases we will do follow-up graph-walk queries like:

  ```sql
  LOOKUP ON player \
    WHERE player.name == "Kobe Bryant"\
    YIELD id(vertex) AS VertexID, properties(vertex).name AS name |\
    GO FROM $-.VertexID OVER serve \
    YIELD $-.name, properties(edge).start_year, properties(edge).end_year, properties($$).name;
  ```

- Or in MATCH query like this, under the hood, v will be searched on index and v2 will be walked by default graph data structure without involving index.

  ```cypher
  MATCH (v:player{name:"Tim Duncan"})-->(v2:player) \
    RETURN v2.player.name AS Name;
  ```



## Recap

Finally, Let's Recap

- INDEX is sorting PROP DATA to find data on given PURE PROP CONDITION
- INDEX is **not** for Graph Walk
- INDEX is left match, **not** for full-text search
- INDEX has cost on WRITE
- Remember to REBUILD after CREATE INDEX on existing data



Happy Graphing!

Feture image credit to [Alina](https://unsplash.com/photos/ZiQkhI7417A)

