# 基于 NebulaGraph 图数据库的 ID Resolution 方法与代码示例


> 本文是一个基于 NebulaGraph 上的图算法、图数据库、图神经网络的 ID-Mapping 方法综述，除了基本方法思想的介绍之外，我还给大家弄了可以跑的 Playground。

> 本文还在撰写中，TBD 的章节还请见谅。

<!--more-->

## 基于图数据库的用户 ID 识别方法

用户 ID 识别，是一个很常见的图技术应用场景，在不同的语境下它可能还被叫做 Entity Correlation（实体关联）、Entity Linking（实体链接）、ID Mapping（身份映射）等等。ID 识别在大体上做的事情就识别出相同用户在同一个系统或者不同系统中的不同账号。

由于 ID 识别天然地是一个关联关系问题，也是一个典型的图、图数据库应用场景。

### 建立图谱

我们从一个最简单、直接的图谱开始，如下边的图结构示意显示，我们定义了点：

- User
- Phone
- Email
- Device
- IP
- Address

在他们之间有很自然的边：

- used_device
- logged_in_from
- has_phone
- has_address

![schema digram](https://raw.githubusercontent.com/wey-gu/identity-correlation-datagen/main/images/schema.svg?refresh=true)

> 注，这份数据是开源的，地址在 https://github.com/wey-gu/identity-correlation-datagen

### 根据确定规则获取 ID Mapping

最简单、直接的方法，在特定的场景下也可能是有用的，试想像 email、IP 地址、上网设备这些有严格结构的数据，在它们成为图谱中的点的时候，简单的相等关系就足以找出这样对应关系，比如：

- 拥有相同的 email
- 使用过相同的 IP 地址
- 使用过相同的设备

在前边的图谱、图数据库中，拥有相同的 email 可以直接表达为如下的图模式（Graph Pattern）。

```
(:user)-[:has_email]->(:email)<-[:has_email]-[:user]
```

下图为顶点： user 与边：has_email 的一个图的可视化结果，可以看到这其中有两个三个点相连的串正是符合拥有相同 email 的模式的点。

> 注
>
> - 这个结果的数据源在 https://github.com/wey-gu/identity-correlation-datagen/tree/main/sample/hand_crafted 
> - 如果通过线上访问本文，你可以鼠标悬停（获取点上的属性）和框选放大每一个点和子图哦。

<iframe src="https://datapane.com/reports/v7JN2E7/user-shared-email/embed/" width="100%" height="540px" style="border: none;">IFrame not supported</iframe>

显然，在构建 ID Mapping 系统的过程中，我们就是通过在图数据库中直接查询，可视化渲染结果来看到等效的洞察，这个查询可以写成：

```cypher
MATCH p=(:user)-[:has_email]->(:email)<-[:has_email]-[:user]
RETURN p limit 10
```

在这个数据上的结果则是：

![](./shared_email.webp)

然而，在更多真实世界中，这样的模式匹配往往不能解决更多稍微复杂一点的情形：

比如从上边的图中我们可以看到这两个匹配了的映射中，`holly@welch.org` 关联下的两个用户的姓名是不同的，而 `veronica.j@yahoo.com` 关联下的两个用户姓名是完全相同的。

```CSV
user_2,holly@welch.org,Holly Pollard,1990-10-19,1 Amanda Freeway Lisaland  NJ 94933,600-192-2985x041
user_21,holly@welch.org,Holly,0000-10-19,1 Amanda Freeway Lisaland  NJ 94933,(600)-192-2985
```

再比如 `Sharon91@gmail.com` 和 `Sharon91+001@gmail.com` ，这两个人的姓名不同，但是手机和地址也是不同的。

```CSV
user_17,Sharon91@gmail.com,Sharon Mccoy,1958-09-01,1 Southport Street Apt. 098 Westport  KY 85907,(814)898-9079x898
user_18,Sharon91+001@gmail.com,Kathryn Miller,1958-09-01,1 Southport Street Apt. 098 Westport  KY 85907,(814)898-9079x898
```

比较庆幸的是我们只需要增加类似于“拥有相同邮箱”的其他条件就可以把这行庆幸考虑进来了，而随之而来的问题是：

- 不是所有的数据都存在某一个确定条件的相等（图谱中的直接关联关系），如何处理这样的关系？

比如：

```CSV
user_5,4kelly@yahoo.com,April Kelly,1967-12-01,Schmidt Key Lake Charles AL 36174,410.138.1816x98702
user_23,4kelly@yahoo.com,Kelly April,2010-01-01,Schmidt Key Lake Charles AL 13617,410-138-1816
```

- 如何表现 `Sharon91@gmail.com` 与 `Sharon91+001@gmail.com` 的相似性？
- 如何将多种匹配规则的信息都纳入关联系统？

### 基于复合条件量化方法

在前边，我们提到了集中确定规则无法处理的问题，其实，它们都可以归结为这两点。

1. 需要多因素（规则）进行综合考虑与判定
2. 需要对非确定条件（属性）进行处理，挖掘隐含相等、相似的关联关系（边）

对于 1. ，很自然我们可以想到对多种关联条件进行评分（score），按照多种条件的重要程度进行加权，给出认定为关联的总分的阈值。

有了多因素评分的机制，我们只需要考虑如何在确定的多因素基础之上，增加对不确定因素的处理，从而解决 2.的情况。这里，非确定的条件可能是：

a. 表现结构化数据的相似性：`Sharon91@gmail.com` 与 `Sharon91+001@gmail.com`

b. 表现非结构化数据的相似性：

- `Schmidt Key Lake Charles AL 36174` 与 `Schmidt Key Lake Charles AL 13617`
- `600-192-2985x041` 与`(600)-192-2985`

对于 a. 的结构化数据中的相似性，有两个思路是可以考虑的：

- 直接进行两个值的相似度
  - 直接判定子字符串
  - 运算 Jaccard index 等类似的相似度
- 拆分为更细粒的多个属性
  - 将 email `foo+num@bar.com` 拆分成三个子属性 email.id: `foo`, email.alias: `num`, email.domain: `bar.com`
    - 然后就可以设计详细的确定性规则：email.id 相等、甚至再在此基础上应用其他非确定规则
    - 有时候，比如对于 email.domain 字段，我们还知道 gmail.com 和 googlemail.com 是等价的，这里的处理也是可以考虑的（像是`user_19,brettglenn@googlemail.com` 与 `user_8,brettglenn@gmail.com`，但从邮箱判断背后就是同一个持有者)

而对于 b. 的非结构属性相似性距离，处理方式可以根据具体的 domain knowledge 千差万别：

- 像 `Schmidt Key Lake Charles AL 36174` 与 `Schmidt Key Lake Charles AL 13617` 的地址信息，除了可以用值的相似度之外，还可以把它转换成地理类型的属性，比如一个经纬度组成的点，从而计算两个点之间的地理距离，根据给定的距离值来打分。

> 注，你知道吗？NebulaGraph 图数据库中原生支持地理类型的属性与索引，可以直接创建 Point 类型的地理属性，并计算两个 Point 之间的距离。

- 而像 `600-192-2985x041` 与`(600)-192-2985` 这种字符串形式的电话号码，则可以统一转化为`<国家码>+<区域码>+<本地号码>+<分机号>`这样的结构化数据，进一步按照结构化数据的方式处理

#### 基于复合条件量化方法实操

TBD

#### 利用 Active Learning 的方法交互式学习评分权重

TBD

### 基于图算法的方法

前边的方法中我们直接利用了用户的各项属性、行为事件中产生的关系，并利用各种属性、值相似度的方法建立了基于概率或者带有评分的关联关系。而在通过其他方法增加了新的边之后的图上，我们也可以利用图算法的方法来映射潜在的相同用户 ID。

#### 图相似性算法

利用节点相似性图算法，比如 Jaccard、余弦相似度等，我们可以或者利用图库之上的图计算平台全量计算相似度，也可以用图查询语句实现全图或者给点之间的相似度，最后给相似度一定的阈值来帮助建立新的（考虑了涉及边的）映射关系。

> 注，这里的 Jaccard index 和我们前边提到的比较两个字符串的方法本质是一样的，不过我们现在提及的是应用在图上的点之间存在相连点作为算法中的“交集”的实现。

#### 社区发现算法

自然地，还可以用社区发现的算法全图找出给定的基于边之下的社区划分，调试算法，使得目标划分社区内部点为估计的相同用户。

#### 基于图算法方法实操

TBD

##### 基于图查询的 Jaccard 实现

TBD

##### 基于图计算平台的 Jaccard 实操

TBD

##### 基于图社区发现算法的实操

TBD

### 基于图神经网络的方法

我们注意到，在我们讲以上不同的方法相结合的时候，都是把前导方法的结果作为图上的边，进而作为后边的方法的输入，实际上相同用户 ID 的识别本质上就是在图上去预测用户之间链接、边。

而在 GNN 的方法中，除了我们在欺诈检测中利用到的节点分类（属性预测）之外，链接预测（Link Prediction）是另一个常见的算法目标和应用场景。我们自然也可以用 GNN 的方法，结合 1. 非 GNN 方法获得的和 2. 已经有的人为标注的链接，学习、预测图上的 ID 映射。

值得注意的是，GNN 的方法只能利用数字型的 feature、属性，我们没办法把非数字型的属性像在分类情况里那样枚举为数值，相反，我们在真正的 GNN 之前，可以用其他的图方法去建立基于打分、或者相似度的边建立。这时候，这些前边的方法成为了 GNN 链路预测的特征工程。

#### 基于 GNN 的实操

TBD



> Feature image credit by [Cosmin Serbin](https://unsplash.com/photos/2fn_pxLMS9g)

