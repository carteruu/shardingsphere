+++
title = "分页"
weight = 2
+++

完全支持 MySQL、PostgreSQL 和 Oracle 的分页查询，SQLServer 由于分页查询较为复杂，仅部分支持。

## 分页性能

### 性能瓶颈

查询偏移量过大的分页会导致数据库获取数据性能低下，以 MySQL 为例：

```sql
SELECT * FROM t_order ORDER BY id LIMIT 1000000, 10
```

这句 SQL 会使得 MySQL 在无法利用索引的情况下跳过 1,000,000 条记录后，再获取 10 条记录，其性能可想而知。
而在分库分表的情况下（假设分为2个库），为了保证数据的正确性，SQL 会改写为：

```sql
SELECT * FROM t_order ORDER BY id LIMIT 0, 1000010
```

即将偏移量前的记录全部取出，并仅获取排序后的最后 10 条记录。这会在数据库本身就执行很慢的情况下，进一步加剧性能瓶颈。
因为原SQL仅需要传输 10 条记录至客户端，而改写之后的SQL则会传输 `1,000,010 * 2` 的记录至客户端。

### ShardingSphere 的优化

ShardingSphere 进行了 2 个方面的优化。

首先，采用流式处理 + 归并排序的方式来避免内存的过量占用。由于 SQL 改写不可避免的占用了额外的带宽，但并不会导致内存暴涨。
与直觉不同，大多数人认为 ShardingSphere 会将 `1,000,010 * 2` 记录全部加载至内存，进而占用大量内存而导致内存溢出。
但由于每个结果集的记录是有序的，因此 ShardingSphere 每次比较仅获取各个分片的当前结果集记录，驻留在内存中的记录仅为当前路由到的分片的结果集的当前游标指向而已。
按归并思想合并 m 个长度为 n 的已排序数组，时间复杂度为 `O(mn(log m))`，一般分片数量 m 都较小，可以认为时间复杂度为 `O(n)`,性能损耗很小。

其次，ShardingSphere 对仅落至单分片的查询进行进一步优化。
落至单分片查询的请求并不需要改写 SQL 也可以保证记录的正确性，因此在此种情况下，ShardingSphere 并未进行 SQL 改写，从而达到节省带宽的目的。

## 分页方案优化

由于 LIMIT 并不能通过索引查询数据，因此如果可以保证 ID 的连续性，通过 ID 进行分页是比较好的解决方案：

```sql
SELECT * FROM t_order WHERE id > 100000 AND id <= 100010 ORDER BY id
```

或通过记录上次查询结果的最后一条记录的 ID 进行下一页的查询：

```sql
SELECT * FROM t_order WHERE id > 100000 LIMIT 10
```

## 分页子查询

Oracle 和 SQLServer 的分页都需要通过子查询来处理，ShardingSphere 支持分页相关的子查询。

- Oracle

支持使用 rownum 进行分页：

```sql
SELECT * FROM (SELECT row_.*, rownum rownum_ FROM (SELECT o.order_id as order_id FROM t_order o JOIN t_order_item i ON o.order_id = i.order_id) row_ WHERE rownum <= ?) WHERE rownum > ?
```

目前不支持 rownum + BETWEEN 的分页方式。

- SQLServer

支持使用 TOP + ROW_NUMBER() OVER 配合进行分页：

```sql
SELECT * FROM (SELECT TOP (?) ROW_NUMBER() OVER (ORDER BY o.order_id DESC) AS rownum, * FROM t_order o) AS temp WHERE temp.rownum > ? ORDER BY temp.order_id
```

支持 SQLServer 2012 之后的 OFFSET FETCH 的分页方式：

```sql
SELECT * FROM t_order o ORDER BY id OFFSET ? ROW FETCH NEXT ? ROWS ONLY
```

目前不支持使用 WITH xxx AS (SELECT ...) 的方式进行分页。由于 Hibernate 自动生成的 SQLServer 分页语句使用了 WITH 语句，因此目前并不支持基于 Hibernate 的 SQLServer 分页。
目前也不支持使用两个 TOP + 子查询的方式实现分页。

- MySQL, PostgreSQL

MySQL 和 PostgreSQL 都支持 LIMIT 分页，无需子查询：

```sql
SELECT * FROM t_order o ORDER BY id LIMIT ? OFFSET ?
```
