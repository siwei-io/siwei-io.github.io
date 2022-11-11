# 用 dbt 为 Nebulagraph 导入数据




> 如何把相对原始的数据处理、建模并导入 NebulaGraph？本文用一个端到端的示例演示，从多数据源聚合数据，清理、转换成 NebulaGraph 建模的属性图点边记录，最后导入成图谱的全流程。

## 任务

假设作为一个类似于 Netflix、爱奇艺的服务提供商，我们需要利用 NebulaGraph 搭建一个用户-电影知识图谱，来辅助支撑推荐、问答和推荐理由等常见由图谱支撑的场景。

知识图谱需要的数据存在在不同的数据源，比如一些公开的 API、数仓中的不同数据库、静态的文件等。这时候，我们需要以下几个步骤来从数据构建图谱：

- 分析可能获取的数据
- 选取关心的关联关系，图建模
- 抽取关联关系，导入图数据库

## 源数据

假设我们的数据来源是  [OMDB](https://www.omdb.org/en/us/content/Help:DataDownload) 和 [MovieLens](https://grouplens.org/datasets/movielens/)。

OMDB 是一个开放的电影数据库

## 实操

