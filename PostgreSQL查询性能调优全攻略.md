1. 执行计划：

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
