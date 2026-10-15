# 1. 执行计划：

1.1 获取执行计划
使用 EXPLAIN 命令查看执行计划，加上 ANALYZE 会实际执行查询并返回真实耗时（注意：对于写操作，ANALYZE 会真正修改数据，建议在事务内执行并回滚）。

```sql
-- 基础计划（不执行）
EXPLAIN SELECT * FROM orders WHERE order_date > '2025-01-01';

-- 带实际执行信息（推荐）
EXPLAIN (ANALYZE, BUFFERS, COSTS, TIMING, FORMAT TEXT) 
SELECT * FROM orders WHERE order_date > '2025-01-01';
```

```text
关键参数说明：

ANALYZE：真实执行并返回实际时间（毫秒）。
BUFFERS：显示共享缓冲区命中/读取/脏块数量，帮助判断 I/O 压力。
COSTS：显示启动成本、总成本的估算值。
TIMING：显示每个节点的实际启动时间和执行时间。
```

1.2 执行计划节点解读
一个典型的执行计划如下（示例）：

```javascript

Seq Scan on orders  (cost=0.00..48765.23 rows=102300 width=48) (actual time=0.023..123.456 rows=102300 loops=1)
  Filter: (order_date > '2025-01-01'::date)
  Rows Removed by Filter: 897700
  Buffers: shared hit=1000 read=5000
Planning Time: 0.123 ms
Execution Time: 125.678 ms
```


<img width="1009" height="457" alt="image" src="https://github.com/user-attachments/assets/af296cfc-92b9-4644-a010-8e8d08e4935b" />


1.3 常见低效计划模式


<img width="1007" height="328" alt="image" src="https://github.com/user-attachments/assets/308edb0c-04bc-4327-be4d-4969149ef797" />


# 2. 统计信息：优化器的“眼睛”


2.1 查看统计信息

```javascript
-- 查看表的统计信息最近更新时间
SELECT schemaname, tablename, last_analyze, last_autoanalyze 
FROM pg_stat_user_tables 
WHERE tablename = 'orders';

-- 查看列上的高频值（MCV）和直方图边界
SELECT attname, n_distinct, most_common_vals, most_common_freqs, histogram_bounds
FROM pg_stats 
WHERE tablename = 'orders' AND attname = 'order_date';
```

2.2 自动收集与手动干预

自动收集：由 autovacuum 进程触发，默认当表变化量超过 autovacuum_analyze_threshold（默认 50 行）+ autovacuum_analyze_scale_factor（默认 0.1）即 10% 的行数变化时触发。

手动收集：ANALYZE orders; 或 VACUUM ANALYZE orders;。

调优建议：

对于频繁更新的大表，可以调低 autovacuum_analyze_scale_factor（如 0.02），或单独设置表的 autovacuum_analyze_scale_factor。

对于数据分布严重倾斜的列（如订单状态、用户类型），提高 default_statistics_target（默认 100，可设为 200~1000），使直方图和 MCV 更精确。

```bash
-- 为特定列单独设置统计信息目标
ALTER TABLE orders ALTER COLUMN order_status SET STATISTICS 500;
ANALYZE orders;
```

# 3. 索引：性能加速的“核心武器”

PostgreSQL 支持多种索引类型：B-tree、Hash、GiST、SP-GiST、GIN、BRIN 等。选对索引类型比创建索引本身更重要。

# 3.1 B-tree 索引（默认）
适用于等值查询（=）、范围查询（>, <, BETWEEN）、排序（ORDER BY）、以及 LIKE 'prefix%'。

创建时机：

* 查询条件中频繁出现的列。
* 参与 JOIN 的关联列。
* 排序或分组列。


复合索引（多列）：

遵循最左前缀原则：索引 (a, b, c) 可以支持 WHERE a = ?，WHERE a = ? AND b = ?，但不能有效支持 WHERE b = ? 或 WHERE c = ?。

将等值查询列放在左侧，范围查询列放在右侧，以获得最佳性能。

```sql
-- 创建复合索引
CREATE INDEX idx_orders_date_status ON orders(order_date, order_status);

-- 以下查询会使用该索引
EXPLAIN SELECT * FROM orders WHERE order_date > '2025-01-01' AND order_status = 'PAID';
-- 以下查询只能部分使用索引（仅用于 order_date 过滤）
EXPLAIN SELECT * FROM orders WHERE order_date > '2025-01-01';
```


# 3.2 部分索引（Partial Index）

针对表中满足特定条件的子集创建索引，可大幅减小索引体积，提高维护效率。

```sql
-- 只对未完成的订单建立索引
CREATE INDEX idx_orders_pending ON orders(order_id, created_at) 
WHERE order_status != 'COMPLETED';

-- 查询自动利用该部分索引
SELECT * FROM orders WHERE order_status = 'PENDING' AND created_at > now() - interval '7 days';
```

# 3.3 表达式索引（Expression Index）
当查询条件中使用函数或表达式时，普通索引无法使用，需要创建表达式索引

```sql
-- 常见场景：忽略大小写查询
CREATE INDEX idx_users_lower_email ON users(lower(email));

-- 查询将走索引
SELECT * FROM users WHERE lower(email) = 'admin@example.com';
```


# 3.4 GIN 索引（全文检索与 JSONB）
适合数组、全文检索、JSONB 键值查询。

```sql
-- 为 JSONB 字段创建 GIN 索引（支持 ?、?|、?&、@> 等操作符）
CREATE INDEX idx_orders_metadata ON orders USING gin(metadata);

-- 查询
SELECT * FROM orders WHERE metadata @> '{"priority": "high"}';
```

# 3.5 BRIN 索引（块范围索引）

适用于天然有序的字段（如自增 ID、时间戳），且数据物理上连续存储的表。BRIN 索引极小，但扫描块范围时可能产生误报，适合超大表且允许少量误扫的场景。

```sql
CREATE INDEX idx_orders_date_brin ON orders USING brin(order_date) 
WITH (pages_per_range = 128);

-- 查询走 BRIN 索引
SELECT * FROM orders WHERE order_date BETWEEN '2025-06-01' AND '2025-06-30';
```

# 3.6 索引维护与监控

查看索引使用率：通过 pg_stat_user_indexes 获取索引扫描次数和读取次数，长期未使用的索引应考虑删除。

```sql
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY tablename;
```

索引膨胀：使用 pgstattuple 扩展检查索引碎片，定期重建（REINDEX INDEX）可回收空间。

```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;
SELECT * FROM pgstattuple('idx_orders_date_status');
```
