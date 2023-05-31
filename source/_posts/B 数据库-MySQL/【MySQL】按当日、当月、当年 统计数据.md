---
title: 【MySQL】按当日、当月、当年 统计数据
date: 2023-05-31
categories: 
- 数据库-MySQL
---

参考博客：[MYSQL中取当前周/月/季/年的第一天与最后一天\_mysql 取年月的第一天\_cleanfield的博客-CSDN博客](https://blog.csdn.net/cleanfield/article/details/41447585)

## 1. 方案

### 1.1. 优化之前

数据统计需要按照 日、月、年 为维度来统计交易流水，SQL 如下，语句看起来很工整但是却存在致命的性能问题。「前提：`created` 与 `money` 字段具有联合索引」
```sql
SELECT sum(money)  
FROM trade  
WHERE DATE_FORMAT(created, '%Y-%m-%d') = (SELECT DATE_FORMAT(NOW(), '%Y-%m-%d'));  
  
SELECT sum(money)  
FROM trade  
WHERE DATE_FORMAT(created, '%Y-%m') = (SELECT DATE_FORMAT(NOW(), '%Y-%m'));  
  
SELECT sum(money)  
FROM trade  
WHERE DATE_FORMAT(created, '%Y') = (SELECT DATE_FORMAT(NOW(), '%Y'));

SELECT sum(money)  
FROM trade  
WHERE TO_DAYS(created) = TO_DAYS(NOW());
```

### 1.2. 优化思路

`created` 字段明显存在逻辑的运算，所以查询条件不会走索引，所以更改如下。
```sql
SELECT sum(money)  
FROM trade  
WHERE created >= DATE(DATE_FORMAT(NOW(), '%Y%m%d'))  
  AND created < DATE_ADD(DATE(DATE_FORMAT(NOW(), '%Y%m%d')), INTERVAL 1 DAY)  
#   AND DATE(DATE_FORMAT(NOW(), '%Y%m%d')) = DATE_FORMAT(NOW(), '%Y%m%d');

SELECT sum(money)  
FROM trade  
WHERE created >= DATE_ADD(curdate(), interval -day(curdate()) + 1 day)  
AND created < last_day(curdate());

SELECT sum(money)  
FROM trade  
WHERE created >= DATE_SUB(CURDATE(), INTERVAL dayofyear(now()) - 1 DAY)  
  AND created < concat(YEAR(now()),'-12-31');

```

### 1.3. 结论

目的只有一个，让日期字段不参与到计算当中去。

### 1.4. 所用到的函数

```sql
select last_day(curdate());  
--获取当月最后一天。  
select DATE_ADD(curdate(), interval - day(curdate()) + 1 day);  
--获取本月第一天
```

## 2. 有意思的 SQL（同类型）

### 2.1. SQL

```sql
SELECT ifnull(sum(o.pay) / 0.945, 0)  
FROM `zx-order`.orders_detail o  
         LEFT JOIN `zx-order`.goods g ON g.id = o.goods_id  
         LEFT JOIN `zx-user`.company c ON c.id = g.company_id  
         LEFT JOIN `zx-user`.user u ON u.id = c.admin_id  
WHERE (o.state = 10  
    OR o.state = 3)  
  AND DATE_FORMAT(o.pay_time, '%Y%m%d') = DATE_FORMAT(now(), '%Y%m%d')  
  AND u.au_name = 'XXX'  
GROUP BY u.au_name;  
```

### 2.2. Explain 分析

| id | select_type | table | partitions | type   | possible_keys                                                                  | key       | key_len | ref                   | rows  | filtered | Extra                    |
|----|-------------|-------|------------|--------|--------------------------------------------------------------------------------|-----------|---------|-----------------------|-------|----------|--------------------------|
| 1  | SIMPLE      | g     |            | index  | PRIMARY,companyid,idx_goodstype_type                                           | companyid | 5       |                       | 70985 | 100.00   | Using where; Using index |
| 1  | SIMPLE      | c     |            | eq_ref | PRIMARY                                                                        | PRIMARY   | 4       | zx-order.g.company_id | 1     | 100.00   |                          |
| 1  | SIMPLE      | u     |            | eq_ref | PRIMARY,uaccount_anname                                                        | PRIMARY   | 4       | zx-user.c.admin_id    | 1     | 10.00    | Using where              |
| 1  | SIMPLE      | o     |            | ref    | state,goodsid,idx_state_tenantid_orderno_goodsid,idx_state_paytime_pay_goodsid | goodsid   | 5       | zx-order.g.id         | 70    | 50.59    | Using where              |

表结构姑且不管，先看索引。 `o` 表存在字段 `state` 和 `pay_time` 的联合索引，`o` 表字段 `goods_id`，`c` 表字段 `admin_id` ，`u` 表字段 `au_name` 均存在索引。

什么都不修改的情况下，响应时间为 24000 ms 左右。

### 2.3. 初步思路

第一步肯定要修改的查询条件 `u.au_name = '薛庆民' ` ，字段 au_name「 varchar(20) 」，更改为主键查询。通过测试，耗时 1363 ms 左右，虽然快了很多但是还是不满足需求。

`ifnull(sum(o.pay) / 0.945, 0)` 去除不去效果影响不大。

```sql
SELECT ifnull(sum(o.pay) / 0.945, 0)  
FROM `zx-order`.orders_detail o  
         LEFT JOIN `zx-order`.goods g ON g.id = o.goods_id  
         LEFT JOIN `zx-user`.company c ON c.id = g.company_id  
         LEFT JOIN `zx-user`.user u ON u.id = c.admin_id  
WHERE (o.state = 10  
    OR o.state = 3)  
  AND DATE_FORMAT(o.pay_time, '%Y%m%d') = DATE_FORMAT(now(), '%Y%m%d')  
  AND u.id = 201233  
GROUP BY u.id;
```

思路：优先使用主键来查询，而不是索引，避免回表。

##### 2.3.1. AliYun 数据库治理服务给了一个优化方案。

将 `AND DATE_FORMAT(o.pay_time, '%Y%m%d') = DATE_FORMAT(now(), '%Y%m%d')` 替换，把之前日期 = 更改为 `<` 和 `>` 的组合，并取中间值。速度一下子减少到只需要 66 ms 左右就可以查询到结果。

```sql
SELECT IFNULL(SUM(`o`.`pay`) / 0.945, 0)  
FROM `zx-order`.`orders_detail` `o`  
         LEFT JOIN `zx-order`.`goods` `g` ON `g`.`id` = `o`.`goods_id`  
         LEFT JOIN `zx-user`.`company` `c` ON `c`.`id` = `g`.`company_id`  
         LEFT JOIN `zx-user`.`user` `u` ON `u`.`id` = `c`.`admin_id`  
WHERE (`o`.`state` = 10  
    OR `o`.`state` = 3)  
  AND `o`.`pay_time` >= DATE(DATE_FORMAT(NOW(), '%Y%m%d'))  
  AND `o`.`pay_time` < DATE_ADD(DATE(DATE_FORMAT(NOW(), '%Y%m%d')), INTERVAL 1 DAY)  
  AND DATE(DATE_FORMAT(NOW(), '%Y%m%d')) = DATE_FORMAT(NOW(), '%Y%m%d')  
  AND `u`.`au_name` = 'XXX'  
GROUP BY `u`.`au_name`
```

**思路：按日统计的 SQL 更改为范围，去除字段 `pay_time` 上的运算，这样子可让字段 `pay_time` 走索引。**
