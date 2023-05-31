---
title: 【MySQL】大数量下的表结构调整
date: 2023-05-31
categories: 
- 数据库-MySQL
---

参考博客： [https://blog.csdn.net/procrastination/article/details/116056306](https://blog.csdn.net/procrastination/article/details/116056306)

> 大数据量表调整表结构时，有时间会非常慢并且会失败，所以一种比较好的方法是通过类似表修改。
> 
> **正式环境操作数据一定要当心。**

### 1. 创建需要修改表的类似表

```sql
CREATE TABLE vehicle_bak like vehicle;
```

### 2. 修改类似表的表结构和索引

```
可以对比两个表的查询速度，来确认自己加的索引是否合理。
```

### 3. 将 原始表数据 插入到 类似表 中

```sql
insert into vehicle_bak select * from vehicle;
```

### 4. 更改表名

```sql
rename table vehicle to vehicle_1;
rename table vehicle_bak to vehicle;
```

### 5. 数据对比，确定未丢数据

```sql
select count(*) from vehicle;
```

### 6. 确认没问题，删除旧表

```sql
truncate table vehicle_bak;
drop table vehicle_bak;
```