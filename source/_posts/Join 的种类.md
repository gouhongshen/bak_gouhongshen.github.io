---
title: Join 的种类
date: 2022-08-22 14:56:55
tags:
---

join 实际上是连接操作（concatenation），表 r 和 s 之间的 join 可以表示为 $r \bowtie_{\theta} s$，$\theta$ 表示连接的条件，有等于、大于、小于和自然连接几种. 这篇文章不会陷入如何写一条 join SQL 语句的细节中，而是介绍几种不同类型的 join：
> (natural) inner join
> (natural) left outer join
> (natural) right outer join
> (natural) full outer join

<br>

假设有两张表，属性如下：
> student = { ID, name, dept_name }
> takes   = { ID, course_id, semester, year, grade }，

> student 是学生信息表；takes 是学生选课信息表，注意不是每一个学生都有选课信息。


## natural join & join
实际上 natural 是一个条件，它对这几种 join 的效果是一样的，所以不妨以 inner join 为例搞清楚有无 natural 的区别。

现在有个需求：查找出学生的选课信息，用 inner join 可以写出如下的 SQL 语句：
```SQL
select *
from student join takes on student.ID = takes.ID;

其结果是一张表，包含的属性是这样的：
{ student.ID, takes.ID, name, dept_name, course_id, semester, year, grad}

等价于执行下面的 SQL 语句：
select *
from student, takes
where student.ID = takes.ID;
```

natural join 也可以完成这样一个查询，SQL 语句如下：
```SQL
select *
from student natural join takes;

其结果是一张表，包含的属性如下：
{ ID, name, dept_name, course_id, semester, year, grad }
```

<br>

这里可以很明显的看出 natural join 与 inner join 的**区别**：
> * natural join 只会考虑包含相同属性值（属性同时存在于两张表中）的元组。student 和 takes 表都包含 ID 属性，所以 student *natural join* takes 只会连接那些 ID 属性值相同的元组。若有两张表，r 和 s，包含多个相同的属性，假设为 A、B、C，那么 natural join 相当于 where r.A = s.A && r.B = s.B && r.C = s.C，而 inner join 的条件需要指定，没有默认值。
> * natural join 的结果中，相同属性列只会出现一次，而 inner join 会出现两次（如果条件中包含了相同的属性）。

<br>

当然也有一些**相同点**，除了上面的不同，基本都是一样的。如果 select 不指定输出的列，那么在结果表中，先出现条件中的列（若是 natural join，先出现两张表中共有的列），然后是左表中独有的列，最后再是右表独有的列。

## outer join 与 inner join
假设现在我们要查找所有学生的选课情况，那么下面的 SQL 能办到吗：
```SQL
select *
from student natural join takes;

或者
select *
from student join takes on student.ID = takes.ID;
```

**很遗憾，不能**，为什么呢？因为有些学生没有选课，所以在 takes 表中没有它们的信息，上述语句查询出来的结果只包含那些选过课的学生（ID 值同时存在与 student 和 takes 表中），而没有选过课的学生则会被清除。那么有没有方法列出 student 表中所有学生的选课信息呢，就算他们没有选课，那也应该显示出来，让我知道不是。再比如，如果还有一张 course 表，我现在要知道所有课被选的情况，如果采用上述语句连接 takes 表和 course 表也会出现遗漏某些课程的情况（这些课程没有人选）。

上述的需求可以通过 outer join 办到。outer join 的工作方式和 inner join 差不多，只不过保留那些会被 inner join 丢弃的元组。当然了，保留了那些元组后，一些属性列的值缺失，这些值会使用 null 值填充。

举个例子，假如有一个叫做小红的同学没有选课，那么使用 outer join 后，她的选课信息就如下：
> | ID   | name | dept_name | course_id | semester | year | grad |   
> | ---- | ---- | ----      |  ----     | ----     | ---- | ---- |
> |xx|小红|xxxxxxx|null|null|null|null|

<br>

outer join 有以下三种类型：
> left outer join. 只会保留左表会被丢弃的元组

> right outer join. 只会保留右表会被丢弃的元组

> full outer join. 保留两张表中会被丢弃的元组，其结果是上述两种的并集。

<br>

举个例子，如果要找出哪些没有选课的学生，则可以采用如下的 SQL 语句：
```SQL
select ID, name
from student natural left outer join takes
where course_id is null;

和下面的结果等价：
select ID, name
from takes natural right outer join student
where course_id is null;

```

