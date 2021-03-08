| Author | Reviewer | Version | Update Time | link |
| ------ | -------- | ------- | ----------- | ---- |
| 沈刚 | - | v1.0 | 2021-03-08 | - |

# 优化器之逻辑优化

在 TiDB 中， SQL 优化的过程可以分为逻辑优化和物理部分。逻辑优化主要是基于规则的优化，简称 RBO。物理优化会为逻辑查询计划中的算子选择某个具体的实现，需要用到一些统计信息，决定哪一种方式代价最低，所以是基于代价的优化 CBO。

本章会介绍 TiDB 中主要的集中逻辑优化规则。

## 基本逻辑算子介绍

TiDB 中的逻辑算子主要以下几个：

- DataSource：数据源，代表一个源表，select * from t 里面的 t。
- Selection： 代表了相应的过滤条件，select * from t where a = 5 里面的 where a = 5。
- Projection：投影操作，也用于表达式计算， select c, a +b from t 里面的 c 和 a + b就是投影和表达式计算操作。
- Join：两个表的连接操作，select t1.b, t2.c from t1 join t2 on t1.a = t2.a 中的 t1 join t2 on t1.a = t2.a 就是两个表 t1 和 t2 的连接操作。Join 有内连接，左连接，右连接等多种连接方式。
- Sort：就是 select xx from xx order by 里面的 order by
- Aggregation： 在 select sum(xx) from xx group by yy 中的 group by 操作，按某些列分组。分组之后，可能带一些聚合函数，比如 Max/Min/Sum/Count/Average 等
- Apply：用于做子查询

Selection，Projection，Join（简称 SPJ） 是最基本的 3 种算子。

## 列裁剪

列剪裁的基本思想在于

## 分区裁剪

## 关联子查询去关联

## 子查询相关的优化

## Max/Min 消除

## 谓词下推

## TopN 和 Limit 下推

## Join Reorder