---
layout: post
title:  了解MySQL的explain命令
date:   2019-04-22 19:08:00 +0800
categories: 性能调优
tag: MySQL
---

* content
{:toc}


## 一. 预备知识

**阅读本文章前需要掌握MySQL索引的底层数据结构相关知识，可以查看我之前的文章了解。**
* `索引前导列`: 所谓前导列，就是联合索引的第一列或者从第一列开始连续多列的组合。比如通过语句`CREATE INDEX table1_index ON table1 (x, y, z)`创建索引，那么x、xy、xyz都是前导列，而yz，y，z这样的就不是。
* `覆盖索引`: 覆盖索引是select的数据列只用从索引中就能够取得，不必读取数据表，换句话说查询列要被所建的索引覆盖。


## 二. explain命令的作用

* 使用`explain`关键字可以模拟优化器执行SQL语句，知道MySQL是如何处理你的SQL语句的，从而分析你的查询语句的性能瓶颈。
* 在`select`语句前添加`explain`关键字，MySQL会在查询上设置一个标记，在执行查询时，会返回执行计划信息，而不是返回查询语句的执行结果；如果from子句中包含select语句，该子查询依然会被执行，并且子查询的结果会产生临时表。


## 三. explain讲解

#### 需要使用的表
* Actor演员表
```sql
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
```sql
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
```sql
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

    ```sql
    explain select (select 1 from actor limit 1) from film;
    ```
    ![](/styles/images/mysql/2.png)


2. from子句子查询

    ```sql
    explain select id from (select id from film) as der;
    ```
    ![](/styles/images/mysql/3.png)

这条查询语句执行时产生了一张临时表der，外部select引用了这张临时表。

3. union查询
    ```sql
    explain select 1 union all select 1;
    ```
    ![](/styles/images/mysql/4.png)

union执行结果总是放在一个匿名临时表中，因为这种临时表不在SQL中出现，所以它的id是NULL。


#### select_type列
select_type列表示对应的select是简单查询还是复杂查询，如果是复杂查询又是哪种复杂查询。
1. `SIMPLE`: 简单查询，不包含子查询和union查询
    ```sql
    explain select * from film where id = 2;
    ```
    ![](/styles/images/mysql/5.png)

2. `PRIMARY`: 复杂查询中最外层的select

3. `SUBQUERY`: 包含在select中的子查询，不再from子句中

4. `DERIVED`: 包含在from子句中的子查询，MySQL会将结果存放在一个临时表中，也称为派生表
    ```sql
    explain select (select 1 from actor where id = 1) from (select * from film where id = 1) der;
    ```
    ![](/styles/images/mysql/6.png)

5. `UNION`: 在union语句中的第二个和后面的select

6. `UNION RESULT`: 在union匿名临时表中查询的select
    ```sql
    explain select 1 union all select 1;
    ```
    ![](/styles/images/mysql/7.png)


#### table列
* table列表示explain中的select语句正在访问哪张表。
* 当from子句中有子查询时，table列为`<derivenN>`格式，表示当前查询依赖`id=N`的查询，于是先执行`id=N`的查询。
* 当有union时，UNION RESULT的table列的值是`<union1,2>`，其中`1`和`2`表示参与union的select的id。


#### type列
这一列表示查询语句是关联类型还是访问类型，查找数据行记录的大概范围。
type列数值从最优到最差的排序是：
```
system > const > eq_ref > ref > range > index > ALL。
```
在日常的开发中，我们建议确保查询能达到range以上，最好达到ref。

* `NULL`: MySQL能够在优化阶段分解查询语句，在执行阶段不用再访问表或索引。例如，在索引列中选取最小值，可以通过查找索引完成，并不需要在执行时访问数据表。从下面的执行计划结果能看到table列为NULL，说明这条查询语句并没有访问表，而是直接查找索引得出结果。
    ```sql
    explain select min(id) from film; 
    ```
    ![](/styles/images/mysql/8.png)

* `const`: 当我们使用primary key或unique key的所在列与常量进行等值比较时，可以猜到查询结果只能是0条记录或者1条记录，这个结果的记录数是一个可预见常量，因此使用const标识。

* `system`: system是const的一种特例，如果数据表中只有一条记录的话，那么我们查询的结果也最多是一条记录，这时结果记录数也是一个可预见常量，因此用system标识。
    ```sql
    explain extended select * from (select * from film where id = 1) tmp;
    ```
    ![](/styles/images/mysql/9.png)

* `eq_ref`: 当使用关联查询（join）时，如果在关联条件join on中使用的字段是primary key或unique key，就会被标识为eq_ref。
    ```sql
    explain select * from film_actor left join film on film_actor.film_id = film.id; 
    ```
    ![](/styles/images/mysql/10.png)
    如上图中显示的第二行film表的查询记录，因为在join on条件中使用的film.id就是film表的primary key，所以film表的查询计划结果行的type列为eq_ref。

* `ref`: 相比于eq_ref的区别是查询条件中使用的字段是普通索引，或者是联合索引的前导列，下面列出两种出现ref类型的场景：
    * 简单select查询，name是普通索引。
        ```sql
        explain select * from film where name = "film1";
        ```
        ![](/styles/images/mysql/11.png)

    * 关联selevt查询。
        ```sql
        explain select film_id from film left join film_actor on film.id = film_actor.film_id;
        ```
        ![](/styles/images/mysql/12.png)
        idx_film_actor_id是包含film_id和actor_id字段的联合索引，关联查询中使用到了film_actor表联合索引idx_film_actor_id的前导列film_id。

* `range`: 使用索引字段进行范围查找，通常是in()、between...and...、>、>=、<、<=等操作。
    ```sql
    explain select * from actor where id > 1;
    ```
    ![](/styles/images/mysql/13.png)

* `index`: 扫描索引，通常是比ALL快一些，index是从索引中读取数据，ALL从硬盘中读取，
    ```sql
    explain select * from film;
    ```
    ![](/styles/images/mysql/14.png)
    因为film表中所有字段都走了索引，所以MySQL通过扫描索引取到了数据，因此没有再扫描表。

* `ALL`: 全表扫描，意味着MySQL需要从头到尾的查找需要的行，遇到这种情况通常都是需要增加索引进行优化。
    ```sql
    explain select * from actor;
    ```
    ![](/styles/images/mysql/15.png)


#### possible_keys列
这一列显示MySQL分析查询语句时可能使用到的索引。
如果该列显示NULL，表示不使用索引查询，在这种情况下，通常我们都需要检查where子句，通过创建适当的索引优化查询性能。


#### key列
* 这一列显示了MySQL真正执行查询语句时使用到的索引。
如果查询操作没有使用索引，则该列为NULL；
* 通过explain分析查询语句时可能会出现possible_keys列有值，但key列为NULL的情况，原因是表中数据不多，虽然MySQL分析出可能使用哪些索引，但是实际执行时认为索引对本次查询没什么帮助，选择了全表查询。
* 如果想强制使用possible_keys列中分析出的索引，可以在查询中使用force index、反之强制不使用possible_keys列的索引需要使用ignore index。


#### key_len列
* 这一列显示查询时使用的索引的字节数总和，在使用到联合索引时能通过该值分析出使用到了索引中的那些字段。
* key_len的计算规则：
    * 字符串
        * char(n): n为字节长度
        * varchar(n): 需要2个字节用于存储字符串长度，在使用utf-8字符集时，长度计算公式是3n + 2
    * 数值类型
        * tinyint: 1字节
        * smallint: 2字节
        * int: 4字节
        * bigint: 8字节
    * 时间类型
        * date: 3字节
        * timestamp: 4字节
        * datatime: 8字节
    * 如果字段允许为NULL，还需要1字节用于记录是否为NULL
* 索引的最大长度为768字节，当字符串过长时，MySQL会做一个类似左前缀索引的处理，将前半部分的字符提取出来做索引。
* 样例说明：在film_actor表中创建了一个联合索引idx_film_actor_id，它由film_id和actor_id两个int类型字段组成。
    * 执行下面的语句并得到结果
    ```sql
    explain select * from film_actor where film_id = 2;
    ```
    ![](/styles/images/mysql/16.png)
    按照我们上面的计算规则每个int是4字节，结果中key_len=4说明使用了film_id列进行索引查询。
    * 执行另一条语句并得到结果
    ```sql
    explain select * from film_actor where film_id = 2 and actor_id = 1;
    ```
    ![](/styles/images/mysql/17.png)
    结果中key_len=8，刚好是film_id和actor_id字节之和。


#### ref列
这一列显示在key列中列出的每个索引分别和什么类型的操作数做判断。
* const（常量）
    ```sql
    explain select * from film_actor where film_id = 2;
    ```
    ![](/styles/images/mysql/16.png)
    where子句film_id = 2中film_id是联合索引的前导列，2是常量，因此ref=const。
* 字段名
    ```sql
     explain select film_actor.film_id from film left join film_actor on film.id = film_actor.film_id;
    ```
    ![](/styles/images/mysql/18.png)
    我们看图中第二行是film_actor表的结论，关联条件是film.id = film_actor.film_id，其中film_actor.film_id是联合索引前导列，film.id是关联表字段，因此ref=test.film.id（数据库.表名.字段名），


#### rows列
这一列显示的是MySQL估计需要读取和检测的行数，注意这里不是结果集里的行数。


#### Extra列
这一列显示额外信息，常见的值如下:
* `Using index`: 查询的所有列被索引覆盖，并且where筛选条件（如果有where子句）是索引的前导列，这种结果代表了查询性能高效，一般是使用了**覆盖索引**。
    ```sql
    -- film_id列是film_actor表中联合索引idx_film_actor_id的前导列 
    explain select film_id from film_actor where film_id = 1;
    ```
    ![](/styles/images/mysql/19.png)

* `Using where; Using index`: 查询的列被索引覆盖，并且where的筛选条件是索引列之一，但不是前导列，这种情况无法直接通过索引查找方式查询出符合条件的数据。
    ```sql
    explain select film_id from film_actor where actor_id = 1;
    ```
    ![](/styles/images/mysql/21.png)

* `Using where`: 查询的列未被索引全部覆盖，并且where子句的筛选条件也不是索引的前导列。
    ```sql
    explain select * from actor where name = 'a';
    ```
    ![](/styles/images/mysql/20.png)

* `NULL`: 查询的列未被索引覆盖，但是where筛选条件是索引的前导列。可以这么理解，where子句走了索引，但是需要select的数据列不能全部从索引中获取，需要通过索引回表查询。
    ```sql
    explain select * from film_actor where film_id = 1;
    ```
    ![](/styles/images/mysql/22.png)

* `Using index condition`: 与`Using where`类似，查询的列未被索引全部覆盖，where筛选条件是索引的前导列而且是一个范围查找。
    ```sql
    explain select * from film_actor where film_id > 1;
    ```
    ![](/styles/images/mysql/23.png)

* `Using temporary`: MySQL需要创建一张临时表来处理查询，这种情况一般都是需要优化的，可以考虑索引优化。
    我们可以通过分析actor表和film表的name列去重查询进行分析对比，如下:
    * actor表去重查询name:
        ```sql
        explain select distinct name from actor;
        ```
        ![](/styles/images/mysql/24.png)
        因为actor表的name没有加索引，所以MySQL创建了张临时表后再做去重。
    * film表去重查询name:
        ```sql
        explain select distinct name from film;
        ```
        ![](/styles/images/mysql/25.png)
        因为film表的name建立了idx_name索引，所以查询时Extra列显示为Using index，没有创建临时表。

* `Using filesort`: MySQL对查询结果集进行了外部排序，而不是按照索引次序从表里读取行数据。这种情况下MySQL会扫描所出所有符合条件的记录，并记录排序关键字和行指针，然后对关键字排序并按顺序检索出行信息，遇到这种情况需要考虑使用索引优化查询。
    为了方便理解，我们列举两条查询语句对explain结果进行分析对比。
    * 按name排序查询actor表
    ```sql
    explain select * from actor order by name;
    ```
    ![](/styles/images/mysql/26.png)
    * 按name排序查询film表
    ```sql
    explain select * from film order by name;
    ```
    ![](/styles/images/mysql/27.png)
    从上面的执行结果可以看出，按name排序查询actor表时Extra=Using filesort，也就是对查询结果集做了外部排序；但按name排序查询film表时Extra=Using index；相同类型的查询语句，只是查询的表不同，为什么Extra的值不同呢？原因就是film表的name列加了索引，看过我之前文章的朋友应该都知道MySQL索引是一种排好序的数据结构，因此按name排序查询film表数据实际上是按照索引的顺序查询数据的，不需要再进行外部排序。

* Extra是一个很复杂的项，在这里没法全部说明，需要大家自己去了解。