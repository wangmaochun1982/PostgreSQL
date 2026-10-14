主要内存参数优化
* 1. shared_buffers
参数说明：PostgreSQL 用于缓存数据块的共享内存大小

推荐配置：

物理内存的 25%-30%
最大不超过 16GB（过大可能导致性能下降）
对于大内存服务器，建议设置为 8GB-16GB

配置示例：
```sql

ALTER SYSTEM SET shared_buffers = '8GB';


```
调优建议：

过小会导致频繁的磁盘 I/O
过大会占用过多系统内存，影响操作系统缓存
与 effective_cache_size 配合调整



2. work_mem
参数说明：每个查询操作（如排序、哈希）可使用的内存大小

推荐配置：

根据并发查询数量和系统内存调整
计算公式：work_mem = (系统内存 * 0.25) / max_connections
通常设置为 4MB-64MB
配置示例：

```sql
ALTER SYSTEM SET work_mem = '16MB';

```
调优建议：

过小会导致频繁的临时文件创建
过大会导致内存不足，尤其是在高并发场景
可根据查询类型单独调整（使用 ALTER USER 或 ALTER DATABASE）

3. maintenance_work_mem
参数说明：维护操作（VACUUM、CREATE INDEX 等）可使用的内存大小

推荐配置：

物理内存的 5%-10%
最大不超过 2GB
建议设置为 128MB-2GB
配置示例：
```sql

ALTER SYSTEM SET maintenance_work_mem = '512MB';

```
调优建议：

过小会延长维护操作时间
过大会占用过多系统内存
可根据维护操作频率调整

4. effective_cache_size
参数说明：告知优化器可用的系统缓存大小（不实际分配内存）

推荐配置：

物理内存的 50%-75%
用于优化查询执行计划
配置示例：

```sql
ALTER SYSTEM SET effective_cache_size = '16GB';

```
调优建议：

设置过小会导致优化器选择索引扫描而不是顺序扫描
设置过大会导致优化器选择不适合的执行计划
应反映实际可用的系统缓存大小


5. wal_buffers
参数说明：WAL 缓冲区大小
推荐配置：
默认值（-1）自动调整为 shared_buffers 的 1/32，最小 64KB
通常足够，无需手动调整
对于写入密集型场景，可适当增大
配置示例：
```sql
ALTER SYSTEM SET wal_buffers = '16MB';

```


内存相关的其他参数
1. max_connections
参数说明：最大并发连接数

推荐配置：

根据系统资源和应用需求调整
计算公式：max_connections = (系统内存 * 0.8) / (每个连接所需内存)
通常设置为 100-2000
配置示例：

```sql
ALTER SYSTEM SET max_connections = '500';

```
调优建议：

过大的连接数会消耗大量系统资源
建议使用连接池（如 PgBouncer）减少实际连接数
与 work_mem 等参数配合调整


2. temp_buffers
参数说明：每个连接用于临时表的内存大小
推荐配置：
默认 8MB，通常足够
对于大量使用临时表的场景，可适当增大
配置示例：
```sql
ALTER SYSTEM SET temp_buffers = '16MB';
```


3. shared_memory_type
参数说明：共享内存类型
推荐配置：
Linux：mmap 或 sysv
Windows：windows
建议使用默认值
配置示例：
```sql
ALTER SYSTEM SET shared_memory_type = 'mmap';

```

4. huge_pages
参数说明：是否使用大页内存

推荐配置：

可选值：off、on、try
对于大内存服务器，建议设置为 try 或 on
可减少 TLB miss，提高性能
配置示例：


ALTER SYSTEM SET huge_pages = 'try';
注意事项：

需要操作系统支持并配置大页
配置示例（Linux）：

```sql
echo 1024 > /proc/sys/vm/nr_hugepages
```

内存配置最佳实践
1. 内存分配计算公式
对于总内存为 M 的服务器，内存分配建议：

<img width="795" height="340" alt="image" src="https://github.com/user-attachments/assets/30d9faf5-a84f-4218-8cb1-7bd736bd745c" />

2. 不同负载类型的配置建议
OLTP（在线事务处理）：

特点：高并发、短事务
配置建议：
shared_buffers：25%-30% 内存
work_mem：较小值（4MB-16MB）
max_connections：较大值（500-2000）
启用连接池
OLAP（在线分析处理）：

特点：低并发、复杂查询
配置建议：
shared_buffers：30%-40% 内存
work_mem：较大值（32MB-128MB）
max_connections：较小值（50-200）
maintenance_work_mem：较大值（1GB-2GB）
混合负载：

特点：同时包含 OLTP 和 OLAP 工作负载
配置建议：
折中配置各项参数
考虑使用资源组管理不同类型的查询
监控并根据实际情况调整

3. 内存配置验证
检查内存参数：

```sql
-- 查看主要内存参数
SELECT name, setting, unit, short_desc
FROM pg_settings
WHERE name IN ('shared_buffers', 'work_mem', 'maintenance_work_mem', 
               'effective_cache_size', 'wal_buffers', 'max_connections');
```


计算总内存需求：

```sql
-- 估算总内存使用
SELECT 
  (setting::int * pg_size_bytes(unit)) AS shared_buffers_bytes,
  (setting::int * pg_size_bytes(unit)) AS work_mem_bytes,
  (setting::int * pg_size_bytes(unit)) AS maintenance_work_mem_bytes,
  (max_conn.setting::int * work_mem.setting::int * pg_size_bytes(work_mem.unit)) AS total_work_mem_bytes
FROM pg_settings shared_bufs
JOIN pg_settings work_mem ON work_mem.name = 'work_mem'
JOIN pg_settings maint_work_mem ON maint_work_mem.name = 'maintenance_work_mem'
JOIN pg_settings max_conn ON max_conn.name = 'max_connections'
WHERE shared_bufs.name = 'shared_buffers';
```


内存监控与调优
1. PostgreSQL 内置视图
pg_stat_bgwriter：


-- 查看后台写入器统计，了解缓冲区使用情况
SELECT * FROM pg_stat_bgwriter;
pg_stat_database：


-- 查看数据库级别的缓冲区命中率
SELECT datname, 
       blks_read, 
       blks_hit, 
       100.0 * blks_hit / (blks_read + blks_hit) AS hit_ratio
FROM pg_stat_database
WHERE datname = 'your_database';
pg_stat_activity：


-- 查看当前查询的内存使用情况
SELECT pid, usename, datname, query, 
       EXTRACT(EPOCH FROM (now() - query_start))::int AS duration,
       state
FROM pg_stat_activity
WHERE state = 'active';
2. 系统级监控
Linux 系统：


# 查看内存使用情况
free -m

# 查看进程内存使用
top -p $(pgrep -d ',' postgres)

# 查看共享内存段
ipcs -m

# 查看内存详细信息
cat /proc/meminfo
Windows 系统：


:: 查看内存使用情况
tasklist /fi "imagename eq postgres.exe"

:: 使用性能监视器
perfmon
3. 第三方监控工具
Prometheus + Grafana：

使用 postgres_exporter 收集内存指标
配置 Grafana 仪表盘监控内存使用
关键指标：
共享缓冲区命中率
临时文件使用情况
连接数
内存使用率
pgBadger：


# 分析 PostgreSQL 日志，生成内存相关报告
pgbadger /var/log/postgresql/postgresql-15-main.log
pg_stat_monitor：

增强版查询统计扩展
提供更详细的内存使用信息
常见内存问题及解决方案
1. OOM（内存不足）问题
症状：

数据库进程被 OOM 杀手杀死
系统日志中出现 "Out of Memory" 错误
查询执行失败，报错 "memory allocation failed"
解决方案：

调整内存参数，减少内存使用
降低 max_connections 或 work_mem
增加系统内存
使用连接池减少实际连接数
优化查询，减少内存消耗
预防措施：

设置合理的内存参数
监控内存使用情况
为 PostgreSQL 进程设置 OOM 分数调整值
配置示例：

echo -1000 > /proc/$(pgrep postgres -f "-D")/oom_score_adj
2. 内存泄漏
症状：

数据库进程内存使用持续增长
系统内存使用率逐渐升高
最终导致 OOM 或性能下降
解决方案：

升级到最新版本的 PostgreSQL
检查并修复应用程序中的连接泄漏
定期重启数据库（作为临时解决方案）
报告 bug 给 PostgreSQL 社区
3. 内存碎片化
症状：

系统内存使用率高，但可用内存少
频繁的内存分配和释放
性能下降
解决方案：

使用大页内存（huge_pages）
调整内存分配策略
优化查询，减少内存碎片化
考虑使用内存分配器如 jemalloc


