# 漫谈MySQL的Ranking技术

在实际的项目中，我们经常会遇到需要对数据进行排名（非简单排序）的情况，譬如我们有一个学生各个科目的成绩表，要的到一个学生期末考试总成绩的排名表，怎么做呢？简单的按学生分组后计算总分然后再对总分拍个序吗？那如何处理并列第一、并列第三的问题呢？这时候我们需要的是按总成绩排名，即ranking，而非order by。

## 创建测试表以及测试数据



## 通过SQL Variable创建自定义逻辑的排名


### 按总成绩排名，允许并列，不跳排名

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

## 通过rank() over排名

由于rank是一个很基础的计算功能，所以大部分关系型数据库都有内置的rank函数，我们直接用就可以了。MySQL中rank函数的用法...

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