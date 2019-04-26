---
layout: post
title:  MySQL的explain
date:   2019-04-22 19:08:00 +0800
categories: 性能调优
tag: MySQL
---

* content
{:toc}


## explain命令的作用
* 使用`explain`关键字可以模拟优化器执行SQL语句，知道MySQL是如何处理你的SQL语句的，从而分析你的查询语句的性能瓶颈。
* 在`select`语句前添加`explain`关键字，MySQL会在查询上设置一个标记，在执行查询时，会返回执行计划信息，而不是返回查询语句的执行结果；如果from子句中包含select语句，该子查询依然会被执行，并且子查询的结果会产生临时表。


## explain讲解
#### 需要使用的表
* Actor演员表
```SQL
DROP TABLE
IF EXISTS `actor`;

CREATE TABLE `actor` (
    `id` INT (11) NOT NULL,
    `name` VARCHAR (45) DEFAULT NULL,
    `update_time` datetime DEFAULT NULL,
    PRIMARY KEY (`id`)
) ENGINE = INNODB DEFAULT CHARSET = utf8;

INSERT INTO `actor` 
    (`id`, `name`, `update_time`)
VALUES
    (1,'a','2017-12-22 15:27:18'),
    (2,'b','2017-12-22 15:27:18'),
    (3,'c','2017-12-22 15:27:18');
```

* film影片表
```SQL
DROP TABLE
IF EXISTS `film`;

CREATE TABLE `film` (
    `id` INT (11) NOT NULL AUTO_INCREMENT,
    `name` VARCHAR (10) DEFAULT NULL,
    PRIMARY KEY (`id`),
    KEY `idx_name` (`name`)
) ENGINE = INNODB DEFAULT CHARSET = utf8;

INSERT INTO `film` 
    (`id`, `name`)
VALUES
    (3, 'film0'),
    (1, 'film1'),
    (2, 'film2');
```

* film_actor影片演员关联表
```SQL
DROP TABLE
IF EXISTS `film_actor`;

CREATE TABLE `film_actor` (
    `id` INT (11) NOT NULL,
    `film_id` INT (11) NOT NULL,
    `actor_id` INT (11) NOT NULL,
    `remark` VARCHAR (255) DEFAULT NULL,
    PRIMARY KEY (`id`),
    KEY `idx_film_actor_id` (`film_id`, `actor_id`)
) ENGINE = INNODB DEFAULT CHARSET = utf8;

INSERT INTO `film_actor` 
    (`id`, `film_id`, `actor_id`)
VALUES
    (1, 1, 1),
    (2, 1, 2),
    (3, 2, 1);
```

#### 举例讲解

`mysql> explain select * from actor;`

![](/styles/images/mysql/1.png)

#### id列
* id列是select的序列号，id的个数等于SQL语句中select的个数，并且id列的顺序与select出现的顺序一一对应、
* id的值越大，对应的select语句执行优先级越高，也就是越早执行；id相同则从上往下执行，id为NULL是最后执行。
* MySQL中的select语句分为：
    * 简单查询SIMPLE
    * 复杂查询PRIMARY
        * 简单子查询
        * 派生表，指from语句中的子查询
        * union查询

1. 简单子查询

    ```SQL
    explain select (select 1 from actor limit 1) from film;
    ```
    ![](/styles/images/mysql/2.png)


2. from子句子查询

    ```SQL
    explain select id from (select id from film) as der;
    ```
    ![](/styles/images/mysql/3.png)

这个查询语句执行时产生已一张临时表名为der，外部select引用了这张临时表。

3. union查询
    ```SQL
    explain select 1 union all select 1;
    ```
    ![](/styles/images/mysql/4.png)

union执行结果总是放在一个匿名临时表中，因为这种临时表不在SQL中出现，所以它的id是NULL。


#### select_type列
select_type列表示对应的select是简单查询还是复杂查询，如果是复杂查询又是哪种复杂查询。
1. `SIMPLE`: 简单查询，不包含子查询和union查询
    ```SQL
    explain select * from film where id = 2;
    ```
    ![](/styles/images/mysql/5.png)

2. `PRIMARY`: 复杂查询中最外层的select

3. `SUBQUERY`: 包含在select中的子查询，不再from子句中

4. `DERIVED`: 包含在from子句中的子查询，MySQL会将结果存放在一个临时表中，也称为派生表
    ```SQL
    explain select (select 1 from actor where id = 1) from (select * from film where id = 1) der;
    ```
    ![](/styles/images/mysql/6.png)

5. `UNION`: 在union中的第二和以后的select

6. `UNION RESULT`: 在union匿名临时表中查询的select
    ```SQL
    explain select 1 union all select 1;
    ```
    ![](/styles/images/mysql/7.png)


#### table列
* table列表示explain中的select语句正在访问哪张表。
* 当from子句中有子查询时，table列为`<derivenN>`格式，表示当前查询依赖`id=N`的查询，于是先执行`id=N`的查询。
* 当有union时，UNION RESULT的table列的值是`<union1,2>`，其中`1`和`2`表示参与union的select的id。


#### type列
这一列表示查询语句是关联类型还是访问类型，查找数据行记录的大概范围。
type列数值从最优到最差的排序是：system > const > eq_ref > ref > range > index > ALL。
在日常的开发中，我们建议确保查询能达到range以上，最好达到ref。

* `NULL`: MySQL能够在优化阶段分解查询语句，在执行阶段不用再访问表或索引。例如，在索引列中选取最小值，可以通过查找索引完成，并不需要在执行时访问数据表。从下面的执行计划结果能看到table列为NULL，说明这条查询语句并没有访问表，而是直接查找索引得出结果。
    ```SQL
    explain select min(id) from film; 
    ```
    ![](/styles/images/mysql/8.png)

* `const`: 当我们使用primary key或unique key的所在列与常量进行等值比较时，可以猜到查询结果只能是0条记录或者1条记录，这个结果的记录数是一个可预见常量，因此使用const标识。

* `system`: system是const的一种特例，如果数据表中只有一条记录的话，那么我们查询的结果也最多是一条记录，这时结果记录数也是一个可预见常量，因此用system标识。
    ```SQL
    explain extended select * from (select * from film where id = 1) tmp;
    ```
    ![](/styles/images/mysql/9.png)

* `eq_ref`: 当使用关联查询（join）时，如果在关联条件join on中使用的字段是primary key或unique key，就会被标识为eq_ref。
    ```SQL
    explain select * from film_actor left join film on film_actor.film_id = film.id; 
    ```
    ![](/styles/images/mysql/10.png)

* `ref`: 与eq_ref类似，