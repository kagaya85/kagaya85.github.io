---
title: "Mysql和MariaDB中的Rank排序的坑"
date: 2019-04-01T21:30:20+08:00
tags: ["SQL", "Database"]
categories: ["Code"]
featured_image:
draft: false
---

# Mysql和MariaDB中的Rank排序的坑

题目中需要对一个成绩表按考试科目以及学生做排名

为了加快之后其他步骤的查询速度，我想着将所有成绩都添加好排名后建立一个临时表

想法是

* 先将score表中按考试课程升序，考试成绩降序，学号后两位升序做子查询
* 在新表中调用自定义的rank函数添加排名

rank函数是这样写的：

```mysql
/* 临时rank function 要求原表按id score_mark降序 排列好 */
delimiter //
drop function if exists myrank;
create function myrank(score_id int, score_mark float) 
returns int
READS SQL DATA
begin
    if(@pre_score_id is null or @pre_score_id != score_id) then
        set @pre_score_id := score_id;
        set @pre_score_mark := score_mark;
        set @row := 1;
        set @rank := 1;
    else
        set @row := @row + 1;
        if(@pre_score_mark != score_mark or score_mark is null) then
            set @rank := @row;
            set @pre_score_mark = score_mark;
        end if;
    end if;
    return @rank;
end //
delimiter ;
```

完整的临时表建立过程如下，看起来结构很简单，只是对已经排好序的表统计排名：

```mysql
drop table if exists tmp_table_rank;
create table tmp_table_rank as
select score_id, score_stuno, Score_mark, myrank(score_id, Score_mark) as rank
from (
    select * 
    from score
    order by score_id asc, Score_mark desc, right(score_stuno, 2) asc
);
```

写完后在个人的云服务器上测试了一下结果符合预期，但为了保险起见，又在虚拟机上测试了同样数据和同样的代码，结果却**不一样**！！！

？？？

首先怀疑的是`mysql`版本的问题

服务器上使用的`mysql`版本为`mysql  Ver 14.14 Distrib 5.6.37, for linux-glibc2.12 (x86_64) using  EditLine wrapper`

虚拟机上则是`mysql Ver 15.1 5.5.56-mariaDB`

按理说`mariaDB`本质上还是`mysql`，结果却不一样。。。

在询问同学和老师后怀疑是两者在**排序**和**调用函数**的处理上有不同，理论上期望`mysql`先做完子查询的排序后，对排好序的表计算rank，`mysql`确实是这么做的，但是`mariaDB`似乎还没做完子查询的排序就开始调用函数计算rank，导致每个人的成绩和排名对应不上，出现错误

针对这个怀疑，做出了如下修改：

先将子查询放入一个临时表中，再调用该临时表

```mysql
/* 临时table, 按科目添加排名 */
drop table if exists tmp_table_asc;
create table tmp_table_asc as
select * 
from score
order by score_id asc, Score_mark desc, right(score_stuno, 2) asc;

drop table if exists tmp_table_rank;
create table tmp_table_rank as
select score_id, score_stuno, Score_mark, myrank(score_id, Score_mark) as rank
from tmp_table_asc;
```

结果正确了，果然你俩还不是一个东西啊（苦笑

