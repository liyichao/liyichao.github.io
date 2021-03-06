---
layout: post
title: "Spark SQL paper 注解"
category: 
tags: []
---
{% include JB/setup %}

Spark SQL 是近年来 Spark 新增的重要功能，它让人们用熟悉的 SQL 语言去查询和分析存在各个存储上的结构化数据。本文会顺着一个例子，注解 [Spark SQL paper](https://people.csail.mit.edu/matei/papers/2015/sigmod_spark_sql.pdf)。

# LogicalPlan

看下面的例子：

```
1 import org.apache.spark.sql.SparkSession
2 import spark.implicits._
3 val spark = SparkSession.builder().getOrCreate()
4 val df = spark.read.json("examples/src/main/resources/people.json")
5 df.createOrReplaceTempView("people")
6 val sqlDF = spark.sql("SELECT sum(age) FROM people WHERE 1 > 0 GROUP BY name")
7 sqlDF.show()
```

这是一个比较基本的 Spark SQL 程序，我们先来看第 6 句。这是一个 SQL 语句，Spark 会解析出语法树（目前用的是 antlr4），然后转化为 Spark 内部的结构 catalyst LogicalPlan（位于 `catalyst.parser`）。

LogicalPlan 是什么样的呢？可以看下：

```
scala> sqlDF.explain(true)
== Parsed Logical Plan ==
'Aggregate ['name], [unresolvedalias('sum('age), None)]
+- 'Filter (1 > 0)
   +- 'UnresolvedRelation `people`

== Analyzed Logical Plan ==
sum(age): bigint
Aggregate [name#9], [sum(age#8L) AS sum(age)#34L]
+- Filter (1 > 0)
   +- SubqueryAlias people
      +- Relation[age#8L,name#9] json

== Optimized Logical Plan ==
Aggregate [name#9], [sum(age#8L) AS sum(age)#34L]
+- Relation[age#8L,name#9] json

== Physical Plan ==
*HashAggregate(keys=[name#9], functions=[sum(age#8L)], output=[sum(age)#34L])
+- Exchange hashpartitioning(name#9, 200)
   +- *HashAggregate(keys=[name#9], functions=[partial_sum(age#8L)], output=[name#9, sum#36L])
      +- *FileScan json [age#8L,name#9] Batched: false, Format: JSON, Location: InMemoryFileIndex[file:/Users/liyichao/Projects/spark/examples/src/main/resources/people.json], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<age:bigint,name:string>
```

LogicalPlan 是节点类型为 `TreeNode` 的一棵树。图中，`Filter` 是 `Aggregate` 的子 plan，`UnresolvedRelation` 是 `Filter` 的子 plan。Filter，Aggregate，UnresolvedRelation 都是类型为 TreeNode 的树的节点。这棵树表示了这个 SQL 语句。

TreeNode 可以变换成另一个 TreeNode 而其结果不变，这种变换函数叫 rule （`catalyst.rules`）。例如上文中的逻辑计划去掉 `Filter`，其输出并不会收到影响。rule 有什么用呢？可以看下 Spark SQL 的完整执行过程：

![image](/assets/img/spark_sql_basics/spark_sql_plan.png)

# Analysis

SQL 经过解析后得到的是 "Unresolved LogicalPlan"，这个 LogicalPlan 包含 unresolvedalias, unresolvedRelation 等未解析的东西，然后会经过 `catalyst.analysis` analyze，得到的 plan 就如上面代码段中 "Analyzed Logical Plan" 所示。这过程中 LogicalPlan 的变换，就是通过定义一系列 rule 来实现的。

为什么需要 analyze 呢？以 unresolvedRelation 为例，由 SQL 语句是不能得到 "people" 这张表是如何取数据的，而 analyze 就相当于利用系统里已有的信息（例如 `df.createOrReplaceTempView("people")` 就会往系统注册 people 表），把 unresolvedRelation 转变为某具体的 Relation，这个具体的 Relation 包含该表的数据从哪来的相关信息（例如例子中的 Relation 是包含 json 文件文件名等信息）。

# Logical Optimization

这一阶段也是利用 rule，对 SQL 语句进行优化，这部分优化可以借鉴传统 SQL 优化器的一些规则，例如 `predicate pushdown` 等，优化后是：

```
== Optimized Logical Plan ==
Aggregate [name#9], [sum(age#8L) AS sum(age)#34L]
+- Relation[age#8L,name#9] json
```

可见 Filter 被正确地优化掉了。

# Physical Planning

这一步是把 LogicalPlan 变为物理执行计划 `SparkPlan`。SparkPlan 也是以 TreeNode 形式组织的一棵树，它和 LogicalPlan 有什么区别和联系呢？

* 它知道如何得到结果。SparkPlan 包含以下函数：`final def execute(): RDD[InternalRow] `。得到 RDD 后运行 RDD.compute 就可以得到结果了。
* 也是利用 rule 实现的，具体转换策略在 `core.execution.SparkStrategies`。

# Code Generation

有些 SparkPlan 会生成 Java 字节码（使用了库 [janino](http://janino-compiler.github.io/janino/)），然后在 execute 中调用，以加快执行速度。为什么 Code Generation 会加快执行速度？

以 `(x+y)+1` 为例，其中 x, y 是表的两列。这个语句最后翻译成物理计划会是 `Add(Add(Attribute(x),Attribute(y)),Literal(1))`，而 scala 是不明白 Add 等关键词的，所以执行时会调用 `Add.execute`，其内容可以使 `return left + right`，而 left 又是一个 Add，这样，整个执行下来就会有很多函数调用，如你能直接生成一段 `(x+y)+1` 的代码让 scala 去执行就能消除几乎所有函数调用，从而加快速度。


# 总结

Spark SQL 论文对 Spark SQL 的原理的介绍比较全面，本文补充了一些实例，可以看做其注解，由于是介绍基本原理，所以没深入具体细节，例如 DataType，DataSet，Expression 等基本概念都未涉及，希望以后能有机会介绍。



