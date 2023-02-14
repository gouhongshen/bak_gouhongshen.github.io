---
title: 查询优化（Query Optimization）
date: 2022-07-26 17:05:07
tags:
---

查询优化是干嘛的？有什么意义？贴上书中原文吧：
> **Query optimization** is the process of selecting the most efficient query-evaluation plan from among the many strategies usually possible for processing a given query, especially if the query is complex. <u>We do not expect users to write their queries so that they can be processed efficiently</u>. Rather, we expect the system to construct a query-evaluation plan that minimizes the cost of query evaluation. This is where query optimization comes into play.

> <u>One aspect</u> of optimization occurs at the relational-algebra level, where the system attempts to find an expression that is equivalent to the given expression, but more efficient to execute. <u>Another aspect</u> is selecting a detailed strategy for processing the query, such as choosing the algorithm to use for executing an operation, choosing the specific indices to use, and so on.

## 一个例子
![](/img/query_optimization/1.png)
假设有这样一个需求：“查找所有音乐学院老师的名字和他们教授的课程名（course title）”。假设该需求对应的关系代数表达式（由 SQL 简单转化而来）为：
> $\Pi_{name, title}(\sigma_{dept-name = Music}(instructor \bowtie (teaches \bowtie \Pi_{course-id, titile}(course))))$

> 上图中左边是该表达式的算子树。

<br>

在上面的表达式中，$instructor \bowtie (teaches \bowtie \Pi_{course-id, title}(course))$ 可能会产生非常大的中间结果，实际上我们只关心 dept-name 为音乐学院的教职工，所以，可以将该表达式做一个等价变换：
> $\Pi_{name, title}((\sigma_{dept-name = Music}(instructor)) \bowtie (teaches \bowtie \Pi_{course-id, titile}(course)))$

> 上图中右边是该表达式的算子树。

<br>

接下来给该算子树中的每一种算子选择一个具体的算法，该表达式就可以成为具体的执行计划，如下图：
![](/img/query_optimization/2.jpg)


由上述例子可知，查询优化主要是将用户编写的查询做一些变换以求得到代价最小的执行计划。一般有如下步骤：
> 1. 根据当前的表达式生成一组逻辑等价的表达式。
> 2. 对于生成的每一个表达式，为其中每个操作选择具体的算法，生成一组执行计划。一般而言，每个操作都可以通过多个算法实现，所以每个表达式可以产生多个执行计划。
> 3. 衡量每一个执行计划的代价，然后选择最小的那一个。一般是通过统计信息，比如表的大小、索引深度等来衡量一个执行计划的代价。

> 在优化的过程中，上述三个步骤通常会交叉重叠。比如，可以继续等价变换第三步选择的最小代价的执行计划的表达式。

## 关系代数表达式的变换
