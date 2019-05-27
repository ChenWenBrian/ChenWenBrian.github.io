# 漫谈MySQL的Ranking技术

在实际的项目中，我们经常会遇到需要对数据进行排名（非简单排序）的情况，譬如我们有一个学生各个科目的成绩表，要的到一个学生期末考试总成绩的排名表，怎么做呢？简单的按学生分组后计算总分然后再对总分排个序吗？那如何处理并列第一、并列第三的问题呢？这时候我们需要的是按总成绩排名，即ranking，而非order by。

## 创建测试表以及测试数据

首先我们创建一个测试表并写入一些测试数据：

```sql
CREATE TABLE `std_score` (
  `id` int(11) NOT NULL,
  `student_id` int(11) DEFAULT NULL,
  `class_id` int(11) DEFAULT NULL,
  `score` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

INSERT INTO `std_score` VALUES 
(1,1,1,99),
(2,1,2,90),
(3,1,3,80),
(4,2,1,99),
(5,2,2,90),
(6,2,3,80),
(7,3,1,80),
(8,3,2,90),
(9,3,3,99),
(10,4,1,87),
(11,4,2,79),
(12,4,3,98),
(13,5,1,87),
(14,5,2,90),
(15,5,3,97),
(16,6,1,78),
(17,6,2,97),
(18,6,3,89),
(19,7,1,67),
(20,7,2,78),
(21,7,3,89),
(22,8,1,62),
(23,8,2,59),
(24,8,3,71);
```

## 通过SQL Variable创建自定义逻辑的排名

按照常规逻辑，我们要按总分的到排名信息，我们会先对学生分组，然后对学生的成绩求和得到总分，然后倒排总分，打完收工。

```sql
SELECT 
    c.student_id, SUM(c.score) AS t
FROM
    std_score c
GROUP BY c.student_id
ORDER BY t DESC
```

| student_id | total |
| ---------- | ----- |
| 5          | 274   |
| 2          | 269   |
| 3          | 269   |
| 1          | 269   |
| 4          | 264   |
| 6          | 264   |
| 7          | 234   |
| 8          | 192   |

但是根据输出的结果，我们发现了2个问题：
1. 虽然按分数从高到低排序了，但是没有名次信息；
2. 总分有重复的；

OK，针对第一个问题，我们很容易想到，哦，我还需要一个排序的列，如果是Oracle或者MySQL 8.0，那么我们就有`row number`可以用了。那如何做一个通用的row number列呢，很简单，我们用SQL Variable试试：

```sql
set @num:=0;

SELECT 
    @num:=@num + 1 as row_num, o.*
FROM
    (SELECT 
        c.student_id, SUM(c.score) AS t
    FROM
        std_score c
    GROUP BY c.student_id
    ORDER BY t DESC) o
```

| row_num | student_id | total |
| ------- | ---------- | ----- |
| 1       | 5          | 274   |
| 2       | 1          | 269   |
| 3       | 2          | 269   |
| 4       | 3          | 269   |
| 5       | 4          | 264   |
| 6       | 6          | 264   |
| 7       | 7          | 234   |
| 8       | 8          | 192   |

现在问题1解决了，我们有了名次信息，但是显然问题2并没有解决，因为我们的名次没有去重，没有并列成绩的准确排名。谁也不希望自己明明分数与别人一样高，但是名次却低一名。所以我们要进一步解决总分相同的成绩。要解决这个问题，就涉及到了一个业务问题：去重后，我们要不要跳过重复的名次呢？譬如这儿有3个并列第二名，那么是否允许有第三名呢？还是直接从第5名开始下一个排名呢？

### 按总成绩排名，允许并列，不跳排名

首先我们按照常规逻辑，允许并列排名，但是不跳跃名次，即重复第2名后，依然从第3名开始继续排名。继续使用SQL Variable方式实现，如下：

```sql
set @num:=1;
set @last:=0;

SELECT 
    (@num:=IF(@last:=o.t > o.t, @num + 1, @num)) AS ranking,
    (@last:=o.t) AS last_score,
    o.*
FROM
    (SELECT 
        c.student_id, SUM(c.score) AS t
    FROM
        std_score c
    GROUP BY c.student_id
    ORDER BY t DESC) o
```

| ranking | last_score | student_id | total |
| ------- | ---------- | ---------- | ----- |
| 1       | 274        | 5          | 274   |
| 2       | 269        | 1          | 269   |
| 2       | 269        | 2          | 269   |
| 2       | 269        | 3          | 269   |
| 3       | 264        | 4          | 264   |
| 3       | 264        | 6          | 264   |
| 4       | 234        | 7          | 234   |
| 5       | 192        | 8          | 192   |

但是一般情况下，`last_score`并不是我们最终需要的数据，仅仅是计算的中间产物，对于有强迫症的我们，如何消除该字段呢？其实只要我们确保IF函数计算前，不对`@last`变量赋值即可，即先进行大小比较，后赋值更新`@last`变量。所以可以将IF函数更改为`IF(@last > @last:=o.total, @num + 1, @num)`。改进后的SQL如下：

```sql
set @num:=1;
set @last:=0;

SELECT 
    (@num:=IF(@last > @last:=o.total,
        @num + 1,
        @num)) AS ranking,
    o.*
FROM
    (SELECT 
        c.student_id, SUM(c.score) AS total
    FROM
        std_score c
    GROUP BY c.student_id
    ORDER BY total DESC) o
```

| ranking | student_id | total |
| ------- | ---------- | ----- |
| 1       | 5          | 274   |
| 2       | 1          | 269   |
| 2       | 2          | 269   |
| 2       | 3          | 269   |
| 3       | 4          | 264   |
| 3       | 6          | 264   |
| 4       | 7          | 234   |
| 5       | 8          | 192   |

其实呢，这个需求，大部分数据库其实已经有内置的函数实现了，我们直接用就可以，参考[通过dense_rank() over排名](#%E9%80%9A%E8%BF%87denserank-over%E6%8E%92%E5%90%8D)

### 按总成绩排名，允许并列，跳过中间排名

但是现实中往往有时候排名是要求跳名次的，即重复3个第2名后，从第5名开始继续排名。继续使用SQL Variable方式实现，如下：

```sql
set @num:=1;
set @duplicates:=1;
set @last:=0;

SELECT 
    (@num:=IF(@last > o.total, @num + @duplicates, @num)) AS ranking,
    (@duplicates:=IF(@last = o.total, @duplicates + 1, 1)) AS duplicates,
    (@last:=o.total) AS last_score,
    o.*
FROM
    (SELECT 
        c.student_id, SUM(c.score) AS total
    FROM
        std_score c
    GROUP BY c.student_id
    ORDER BY total DESC) o
```

| ranking | duplicates | last_score | student_id | total |
| ------- | ---------- | ---------- | ---------- | ----- |
| 1       | 1          | 274        | 5          | 274   |
| 2       | 1          | 269        | 1          | 269   |
| 2       | 2          | 269        | 2          | 269   |
| 2       | 3          | 269        | 3          | 269   |
| 5       | 1          | 264        | 4          | 264   |
| 5       | 2          | 264        | 6          | 264   |
| 7       | 1          | 234        | 7          | 234   |
| 8       | 1          | 192        | 8          | 192   |

同样的，一般情况下，`duplicates`和`last_score`并不是我们最终需要的数据，我们一样可以消灭这些中间字段。但是这次的字段计算顺序比上一个例子的要复杂：`ranking`依然是最先计算，并用到上一条记录中的`@duplicates`和`@last`两个变量，然后`duplicate`又必须在`last_score`前计算，就不太适用于上一个的简化方式了。通用的办法是在该查询外面再嵌套一个`select`查询，并隐藏本次查询的字段，SQL如下：

```sql
set @num:=1;
set @duplicates:=1;
set @last:=0;

SELECT 
    h.ranking, h.student_id, h.total
FROM
    (SELECT 
        (@num:=IF(@last > o.total, @num + @duplicates, @num)) AS ranking,
        (@duplicates:=IF(@last = o.total, @duplicates + 1, 1)) AS duplicates,
        (@last:=o.total) AS last_score,
        o.*
    FROM
        (SELECT 
        c.student_id, SUM(c.score) AS total
    FROM
        std_score c
    GROUP BY c.student_id
    ORDER BY total DESC) o) h
```

| ranking | student_id | total |
| ------- | ---------- | ----- |
| 1       | 5          | 274   |
| 2       | 1          | 269   |
| 2       | 2          | 269   |
| 2       | 3          | 269   |
| 5       | 4          | 264   |
| 5       | 6          | 264   |
| 7       | 7          | 234   |
| 8       | 8          | 192   |

这个需求，大部分数据库也已经有内置的函数实现了，我们直接用就可以，参考[通过rank() over排名](#%E9%80%9A%E8%BF%87rank-over%E6%8E%92%E5%90%8D)

## 通过rank() over排名

由于rank是一个很基础的计算功能，所以大部分关系型数据库都有内置的rank函数，我们直接用就可以了。rank()函数在排名时，是会跳跃排名的，MySQL中rank函数的用法如下：

```sql
SELECT 
	rank() over (order by o.total desc) ranking,
    o.*
FROM
    (SELECT 
        c.sid, SUM(c.score) AS total
    FROM
        std_score c
    GROUP BY c.sid) o
```

| ranking | student_id | total |
| ------- | ---------- | ----- |
| 1       | 5          | 274   |
| 2       | 1          | 269   |
| 2       | 2          | 269   |
| 2       | 3          | 269   |
| 5       | 4          | 264   |
| 5       | 6          | 264   |
| 7       | 7          | 234   |
| 8       | 8          | 192   |

## 通过dense_rank() over排名

dense_rank() 是rank不跳排名的一种实现，参考如下：

```sql
SELECT 
	dense_rank() over (order by o.total desc) ranking,
    o.*
FROM
    (SELECT 
        c.sid, SUM(c.score) AS total
    FROM
        std_score c
    GROUP BY c.sid) o
```

| ranking | student_id | total |
| ------- | ---------- | ----- |
| 1       | 5          | 274   |
| 2       | 1          | 269   |
| 2       | 2          | 269   |
| 2       | 3          | 269   |
| 3       | 4          | 264   |
| 3       | 6          | 264   |
| 4       | 7          | 234   |
| 5       | 8          | 192   |