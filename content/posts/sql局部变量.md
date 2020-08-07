---
title: "SQL变量总结"
date: 2019-04-07T21:30:20+08:00
tags: ["SQL", "Database"]
categories: ["Code"]
featured_image:
draft: false
---

# SQL变量总结

### 变量分类

* 用户变量
  * 以`@`开始，形式为`@变量名`
  * 全局变量
    * `set global 变量名` 或者 `set @@global.变量名`
    * 对所有客户端生效
    * 只有具有super权限才可以设置全局变量
  * 会话变量
    * 用户变量与`mysql`客户端绑定，只对当前用户使用的客户端生效
    * 只对连接的客户端生效
* 局部变量
  * `declare`专门用于声明
  * 作用范围在`begin`和`end`语句块之间

### 定义方法

* "=",如 set @a =3,@a:=5
* ":="。select常常这样使用
* set可以使用以上两种形式设置变量。而select只能使用":="的形式设置变量
* 未定义的变量初始化是null

### 示例题目：

X 市建了一个新的体育馆，每日人流量信息被记录在这三列信息中：**序号** (id)、**日期** (date)、 **人流量** (people)。

请编写一个查询语句，找出高峰期时段，要求连续三天及以上，并且每天人流量均不少于100。

| id   | visit_date | people |
| ---- | ---------- | ------ |
| 1    | 2017-01-01 | 10        |
| 2    | 2017-01-02 | 109       |
| 3    | 2017-01-03 | 150       |
| 4    | 2017-01-04 | 99        |
| 5    | 2017-01-05 | 145       |
| 6    | 2017-01-06 | 1455      |
| 7    | 2017-01-07 | 199       |
| 8    | 2017-01-08 | 188   |

对于上面实例数据，应当输出

| id   | visit_date | people |
| ---- | ---------- | ------ |
| 5    | 2017-01-05 | 145       |
| 6    | 2017-01-06 | 1455      |
| 7    | 2017-01-07 | 199       |
| 8    | 2017-01-08 | 188       |



利用**会话变量**+**巧用排序**实现，利用两个中间表逐渐接近目标（个人偏向使用这种）

```mysql
select id, visit_date, people
from (
    select id, visit_date, people, if((days > 0 and @ok = 1) or days >= 3, @ok:=1, @ok:=0) as ok
    from (
        select id, visit_date, people, if(people >= 100, @days:=@days+1, @days:=0) as days
        from stadium, (select @days := 0) as t1
    ) countedDays, (select @ok := 0) as t2
   order by id desc
) oklist
where ok = 1
order by id asc;
```



再提供一种思路：

利用多表联合查询，分为三种情况顺序，思路十分巧妙，代码短小精悍，distinct巧妙的防止了重复选择

```mysql
select distinct a.* from stadium a,stadium b,stadium c
where a.people>=100 and b.people>=100 and c.people>=100
and (
     (a.id = b.id-1 and b.id = c.id -1) or
     (a.id = b.id-1 and a.id = c.id +1) or
     (a.id = b.id+1 and b.id = c.id +1)
) order by a.id
```

> by Derrick