# Access path selection in a relational database management system

## 概述
[这篇文章](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.71.3735&rep=rep1&type=pdf)是最早介绍关系型数据库优化器访问路径选择的文章，很多开源数据库系统，甚至商业数据库的优化器均基于该文章的思路构建，这类优化器可以统称为 System-R 类的优化器。

文章介绍了 IBM 的实验性质数据库系统 System R 的一种基于动态规划算法、自底向上查找最优访问路径的方案，本文对论文内容做简要概括。

1970 年 E.F Codd 在他的论文 A Relational Model of Data for Large Shared Data Banks 中提出关系数据模型；1975 年 IBM 开始研发 System R 关系数据库系统。用户通过 SQL 仅告知系统需要什么数据，并不会告知如何获取这些数据，而优化器则选择表的连接顺序和访问方式，从众多可能中选择代价最小的一种可能。

## SQL 处理流程
SQL 处理包括四个步骤：
* parsing：check correct syntax
* optimization
    * 从 catalog 中获取表和列信息，以及统计信息
    * 选择代价最小的路径
    * 代价最小的解决方案是用一个结构化的 parse tree 来表示的
    * 执行计划用 ASL（Access Specification Lauguage） 来表示
* code generation
    * 一个以表驱动的程序，将 ASL 树转换为机器语言
* execution

## RSS（Research Storage System）
RSS 是 System R 的存储子系统，负责表的物理存储，访问路径，锁、日志和恢复。RSS 提供一个面向 tuple 的接口。

Relation 以元组的集合存储在 RSS 中，元组存储在 4K 的页中，没有元组跨页。页被组织成称之为 segement 的单元，segment 可能包含一个或多个 relation，但没有 relation 可以跨 segment。来自两个或多个 relation 的元组可能出现在相同的 page 中。

访问元组的主要方法是通过 RSS 的 scan 接口。一次 scan 根据给定的 access path 一次返回一个元组。OPEN，NEXT 和 CLOST 是主要的 scan 命令。

两类 Scan：
* segment scan，遍历 segment 获取给定 relation 的所有元组，segment 中所有非空页都会被 touch，然而每个页只会被 touch 一次
* index scan，B-tree 索引，索引页相连，每个索引页被 touch 一次，但数据页可能被检测多次。对于 cluster index（聚簇索引），数据页和索引页都只会被 touch 一次。

无论 segment scan 还是 index scan，都可能带有一组谓词，称之为 search arguments（或 SARGS），这些谓词用于判断一个 tuple 是否需要返回。并非所有的谓词都可以转换为 SARGS，SARGS 是一个布尔表达式，或者布尔表达式的析取范式。

## 单表访问路径代价
单表访问的代价估计，Cost = Page Fetches + W*(RSI Calls)

每个关系 T 的统计信息：
* NCARD(T)，基数
* TCARD(T)，segment 中 page 的数量
* P(T)，segment 中 T 的数据页的占比
* ICARD(I)，索引 I 中不同 key 的数量
* NINDX(I)，索引 I 的页数

这些统计信息在关系加载和搜因创建的时候初始化；一个 UPDATE STATISTICS 命令定期更新这些统计信息。由于需要持有锁，并不会在每次 INSERT，DELETE 和 UPDATE 时都更新统计信息。

优化器使用这些统计信息为每个布尔表达式计算 selectivity factor 'F'，公式如下：
```
column = value
F = 1 / ICARD(column index) if there is an index on column
This assumes an even distribution of tuples among the index key values.
F = 1/10 otherwise

column1 = column2
F = 1/MAX(ICARD(column1 index), ICARD(column2 index))
if there are indexes on both column1 and column2
This assumes that each key value in the index with the smaller cardinality has a
matching value in the other index.
F = 1/ICARD(column-i index) if there is only an index on column-i
F = 1/10 otherwise

column > value (or any other open-ended comparison)
F = (high key value - value) / (high key value - low key value)
Linear interpolation of the value within the range of key values yields F if the column is an arithmetic type and value is known at access path selection time.
F = 1/3 otherwise (i.e. column not arithmetic)
There is no significance to this number, other than the fact that it is less selective than the guesses for equal predicates for which there are no indexes, and that
it is less than 1/2. We hypothesize that few queries use predicates that are satisfied by more than half the tuples.

column BETWEEN value1 AND value2
F = (value2 - value1) / (high key value - low key value)
A ratio of the BETWEEN value range to the entire key value range is used as the
selectivity factor if column is arithmetic and both value1 and value2 are known at
access path selection.
F = 1/4 otherwise
Again there is no significance to this choice except that it is between the default
selectivity factors for an equal predicate and a range predicate.

column IN (list of values)
F = (number of items in list) * (selectivity factor for column = value)
This is allowed to be no more than 1/2.

columnA IN subquery
F = (expected cardinality of the subquery result) / (product of the cardinalities of
all the relations in the subquery’s FROM-list).
The computation of query cardinality will be discussed below. This formula is derived
by the following argument:
Consider the simplest case, where subquery is of the form “SELECT columnB FROM relationC ...”. Assume that the set of all columnB values in relationC contains the set
of all columnA values. If all the tuples of relationC are selected by the subquery,
then the predicate is always TRUE and F = 1. If the tuples of the subquery are
restricted by a selectivity factor F’, then assume that the set of unique values in
the subquery result that match columnA values is proportionately restricted, i.e. the
selectivity factor for the predicate should be F’. F’ is the product of all the subquery’s selectivity factors, namely (subquery cardinality) / (cardinality of all possible subquery answers). With a little optimism, we can extend this reasoning to
include subqueries which are joins and subqueries in which columnB is replaced by an
arithmetic expression involving column names. This leads to the formula given above.

(pred expression1) OR (pred expression2)
F = F(pred1) + F(pred2) - F(pred1) * F(pred2)

(pred1) AND (pred2)
F = F(pred1) * F(pred2)
Note that this assumes that column values are independent.

NOT pred
F = 1 - F(pred) 
```

`interesting order`：如果元组的排序正好跟 GROUP BY 或者 ORDER By 的排序相同，则称元组的排序为 `interesting order`。

对于单表查询，只需要查找代价最小的路径（没有`interesting order`）和带有`interesting order`的代价最小的路径。

不同情况下，计算代价的公式如下：

|SITUATION|COST(in pages)|
|:---------|:---------------------|
Unique index matching an equal predicate |1+1+W
Clustered index I matching one or more boolean factors|F(preds) * (NINDX(I) + TCARD) + W * RSICARD
Non-clustered index I matching one or more boolean factors|F(preds) * (NINDX(I) + NCARD) + W * RSICARD or F(preds) * (NINDX(I) + TCARD) + W * RSICARD if this number fits in the System R buffer 
Clustered index I not matching any boolean factors|(NINDX(I) + TCARD) + W * RSICARD
Non-clustered index I not matching any boolean factors|(NINDX(I) + NCARD) + W * RSICARD or (NINDX(I) + TCARD) + W * RSICARD if this number fits in the System R buffer
Segment scan | TCARD/P + W * RSICARD

## Join 的路径选择
Join 的两张表分为内表和外表。考虑两种连接方式：nested loops 和 merging scans。

连接顺序的可能性很多，通过启发式方法减少连接顺序选择空间。
* 优先考虑有连接谓词表先Join
* 考虑排序， 推导 equivalence class

## 嵌套查询
子查询分类：
*  correlation 子查询，子查询需要根据父查询的元组进行重新评估（re-evaluated），文中提出通过缓存re-evaluated结果，避免相同值的重复 re-evaluated。
*  非 correlation 子查询，子查询结果可以直接获取到，其结果作为父查询的常量

