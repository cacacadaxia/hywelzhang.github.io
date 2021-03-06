---
Author: Hywel
layout: post
title: SparkSQL catalyst工作原理
description: 
date: 星期五, 18. 八月 2017  8:47下午
categories: Spark 
---
> 本文参考http://hbasefly.com/2017/03/01/sparksql-catalyst/完成，写的非常通俗易懂，表示感谢。这是根据这篇博文的学习记录，想深入的推荐去看原文。

对于许多的指标计算，使用Spark最多的场景估计就是SparkSQL了。运行速度远远超过Hive，且上手容易，仅使用SQL就能完成开发。SparkSQL可以通过调用sql("sql语句")这样纯SQL的方式和使用DataSet API这两种方式进行开发。当然这两种方式在底层都是使用的同一个catalyst优化器，所以并没有多少区别，可以根据自己的熟悉程度进行选择。我们今天就来看看Spark运行一个SQL以后是怎样工作的。

整个流程如下图：
![SparkSQL执行流程](/assets/image/postImg/Spark/sparkSQL-catalyst.png)

## Parser（SQL->Unresolved Logical Plan）
首先，Parser模块会将SQL字符串切分成一个一个的Token，根据语法规则构建一颗语法树，这棵树就是上面的Unresolved Logical Plan。Parser模块目前基本都是使用第三方类库ANTLR实现，Hive和SparkSQL一样都是使用这个类库进行语法匹配。
![SparkSQL执行流程](/assets/image/postImg/Spark/sparkSQL-Parser)

## Analyzer(Unresolved Logical Plan -> Logical Plan)
前一个阶段构建了一颗语法树，但是系统并不知道表名，表字段，sum函数等是什么。所以Analyzer阶段，就是将基本的元数据与语法树进行绑定。最重要的元数据信息主要包括两部分：表的Scheme和基本函数信息，表的scheme主要包括表的基本定义（列名、数据类型）、表的数据格式（Json、Text）、表的物理位置等，基本函数信息主要指类信息。

Analyzer会再次遍历整个语法树，对树上的每个节点进行数据类型绑定以及函数绑定，比如people词素会根据元数据表信息解析为包含age、id以及name三列的表，people.age会被解析为数据类型为int的变量，sum会被解析为特定的聚合函数，如下图所示：
![SparkSQL执行流程](/assets/image/postImg/Spark/sparkSQL-Analyzer)

## Optimizer(Logical Plan -> Optimized Logical Plan)
优化器是整个Catalyst核心，主要包括基于规则优化和基于代价优化两种。

常用的三种基于规则优化的方式：谓词下推（Predicate Pushdown）、常量累加（Constant Folding）和列值裁剪（Column Pruning）。
1. 谓词下推：
![谓词下推](/assets/image/postImg/Spark/sparkSQL-PredicatePushdown)
原本是先join，后filter。对于这种操作，可以先filter减少数据量再join，提升效率。谓词下推就是将一些运算等价下移，减少后续操作数据量，提升运行速度。

2. 常量累加：
将一些SQL中可以累加的常量提前累加，避免每次都会重复进行常量累加

3. 列值裁剪：
扫描结果列，自动裁剪一些无关的列，这样可以显著减少数据量，降低网络IO（话说我现在程序里还会手动先select筛选一遍需要的列，好像这操作多余了）
![列值裁剪](/assets/image/postImg/Spark/sparkSQL-ColumnPruning)

（基于代价优化暂无）

## Physical Plan
上述步骤结束生成Optimized Logical Plan以后，逻辑上已经可以开始运行。但是，现在还并不知道对应的系统方法，所以这一步需要将逻辑执行计划转换为物理执行计划。比如Join算子，Spark根据不同场景为该算子制定了不同的算法策略，有BroadcastHashJoin、ShuffleHashJoin以及SortMergeJoin等（可以将Join理解为一个接口，BroadcastHashJoin是其中一个具体实现），物理执行计划实际上就是在这些具体实现中挑选一个耗时最小的算法实现，这个过程涉及到基于代价优化策略。
![物理执行计划](/assets/image/postImg/Spark/sparkSQL-PhysicalPlan)

<font color="pink" size="20">Stay Hungry, Stay Foolish !</font>
