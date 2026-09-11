# 核心概念
postgresql.conf是PostgreSQL数据库的核心配置文件，包含了影响数据库性能、安全性和可靠性的关键参数。PostgreSQL核心参数主要涉及以下核心概念：

```text

内存配置：控制数据库使用的内存资源

WAL配置：控制预写式日志的生成和管理

连接配置：控制数据库连接的数量和属性

检查点配置：控制检查点的执行频率和行为

日志配置：控制数据库日志的生成和格式

自动清理配置：控制autovacuum进程的行为

查询优化配置：控制查询优化器的行为

安全配置：控制数据库的安全性设置

```

# 内存配置参数

内存配置是PostgreSQL性能优化的核心，合理配置内存参数可以显著提高数据库性能。

```text
1. shared_buffers

描述：PostgreSQL用于缓存数据块的内存大小 默认值：128MB 推荐值：系统内存的25% 配置示例：

shared_buffers = 2GB  # 对于8GB内存的服务器

2. work_mem
描述：每个查询操作（如排序、哈希表）可使用的内存大小 默认值：4MB 推荐值：根据并发连接数和查询复杂度调整，通常为64MB-256MB 配置示例：


work_mem = 64MB  # 对于OLTP应用


3. maintenance_work_mem
描述：维护操作（如VACUUM、CREATE INDEX）可使用的内存大小 默认值：64MB 推荐值：系统内存的10%-20%，最大值不超过2GB 配置示例：


maintenance_work_mem = 1GB  # 对于8GB内存的服务器

4. effective_cache_size
描述：PostgreSQL查询优化器认为可用的操作系统缓存大小 默认值：4GB 推荐值：系统内存的50%-75% 配置示例：


effective_cache_size = 6GB  # 对于8GB内存的服务器

5. wal_buffers
描述：WAL（预写式日志）缓冲区大小 默认值：-1（自动设置为shared_buffers的1/32，最大64MB） 推荐值：16MB-64MB 配置示例：


wal_buffers = 16MB

```


# WAL配置参数

```text
WAL配置影响数据库的可靠性和写入性能。

1. wal_level
描述：控制WAL日志的详细程度 默认值：replica 可选值：minimal, replica, logical 推荐值：

replica：用于物理复制
logical：用于逻辑复制 配置示例：

wal_level = replica  # 用于主从复制

2. checkpoint_timeout
描述：检查点之间的最长时间间隔 默认值：5min 推荐值：15min-30min 配置示例：


checkpoint_timeout = 30min

3. max_wal_size
描述：触发检查点的WAL大小上限 默认值：1GB 推荐值：4GB-8GB 配置示例：


max_wal_size = 4GB

4. min_wal_size
描述：检查点后保留的最小WAL大小 默认值：80MB 推荐值：1GB-2GB 配置示例：


min_wal_size = 1GB

5. checkpoint_completion_target
描述：检查点完成目标，控制检查点的持续时间 默认值：0.5 推荐值：0.9 配置示例：


checkpoint_completion_target = 0.9

6. wal_compression
描述：是否压缩WAL日志 默认值：off 推荐值：on（PostgreSQL 14+） 配置示例：


wal_compression = on

```

# 连接配置参数

```text
连接配置控制数据库的连接数量和属性。

1. max_connections
描述：允许的最大并发连接数 默认值：100 推荐值：根据服务器资源和应用需求调整，通常为100-500 配置示例：


max_connections = 3000

2. listen_addresses
描述：PostgreSQL监听的IP地址 默认值：localhost 推荐值：

'*'：监听所有地址
特定IP：监听指定IP地址 配置示例：

listen_addresses = '*'  # 允许所有IP连接

3. port
描述：PostgreSQL监听的端口 默认值：5432 配置示例：


port = 5432

4. superuser_reserved_connections
描述：为超级用户预留的连接数 默认值：3 推荐值：3-5 配置示例：


superuser_reserved_connections = 5

5. tcp_keepalives_idle
描述：TCP连接空闲超时时间 默认值：7200s（2小时） 推荐值：60s-300s 配置示例：


tcp_keepalives_idle = 60s

```


# 日志配置参数

```text
日志配置控制数据库日志的生成和格式。

1. log_min_duration_statement
描述：记录执行时间超过指定毫秒数的SQL语句 默认值：-1（不记录） 推荐值：500ms-1000ms 配置示例：


log_min_duration_statement = 500ms  # 记录执行时间超过500ms的SQL

2. log_line_prefix
描述：日志行前缀格式 默认值：'%m [%p] ' 推荐值：包含时间、进程ID、用户名、数据库名等信息 配置示例：


log_line_prefix = '%m [%p] %q%u@%d %a '  # 包含更多上下文信息

3. log_destination
描述：日志输出目标 默认值：stderr 推荐值：stderr, csvlog 配置示例：


log_destination = 'stderr,csvlog'

4. logging_collector
描述：是否启用日志收集器 默认值：off 推荐值：on 配置示例：


logging_collector = on

5. log_directory
描述：日志文件存储目录 默认值：'log' 配置示例：


log_directory = 'pg_log'

6. log_filename
描述：日志文件名格式 默认值：'postgresql-%Y-%m-%d_%H%M%S.log' 配置示例：


log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'

7. log_rotation_age
描述：日志文件轮换时间间隔 默认值：1d 推荐值：1d 配置示例：


log_rotation_age = 1d

8. log_checkpoints
描述：是否记录检查点信息 默认值：off 推荐值：on 配置示例：


log_checkpoints = on

```

1. 基础配置示例
   
```conf
# 内存配置
shared_buffers = 2GB
work_mem = 64MB
maintenance_work_mem = 1GB
effective_cache_size = 6GB
wal_buffers = 16MB

# WAL配置
wal_level = replica
checkpoint_timeout = 30min
max_wal_size = 4GB
min_wal_size = 1GB
checkpoint_completion_target = 0.9
wal_compression = on

# 连接配置
max_connections = 200
listen_addresses = '*'
port = 5432

# 日志配置
log_min_duration_statement = 500ms
log_line_prefix = '%m [%p] %q%u@%d %a '
log_destination = 'stderr,csvlog'
logging_collector = on
log_directory = 'pg_log'

# 自动清理配置
autovacuum = on
autovacuum_max_workers = 6
autovacuum_vacuum_scale_factor = 0.02
autovacuum_analyze_scale_factor = 0.01

# 查询优化配置
random_page_cost = 1.1  # SSD存储
seq_page_cost = 1.0
```

2. 生产环境配置示例（8GB内存）

```conf

# 内存配置
shared_buffers = 2GB          # 25% of 8GB
work_mem = 64MB              # 每个查询操作可使用的内存
maintenance_work_mem = 1GB    # 12.5% of 8GB
 effective_cache_size = 6GB    # 75% of 8GB
wal_buffers = 16MB            # 适当大小的WAL缓冲区

# WAL配置
wal_level = replica
checkpoint_timeout = 30min
max_wal_size = 4GB
min_wal_size = 1GB
checkpoint_completion_target = 0.9
wal_compression = on

# 连接配置
max_connections = 200
listen_addresses = '*'
port = 5432

# 日志配置
log_min_duration_statement = 500ms
log_line_prefix = '%m [%p] %q%u@%d %a '
log_destination = 'stderr,csvlog'
logging_collector = on
log_directory = 'pg_log'
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0
log_autovacuum_min_duration = 500ms

# 自动清理配置
autovacuum = on
autovacuum_max_workers = 6
autovacuum_vacuum_scale_factor = 0.02
autovacuum_analyze_scale_factor = 0.01
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = -1

# 查询优化配置
random_page_cost = 1.1  # SSD存储
seq_page_cost = 1.0
cpu_tuple_cost = 0.01
cpu_index_tuple_cost = 0.005
cpu_operator_cost = 0.0025

```


3. 生产环境配置示例（32GB内存）

```conf
# 内存配置
shared_buffers = 8GB          # 25% of 32GB
work_mem = 128MB             # 每个查询操作可使用的内存
maintenance_work_mem = 2GB    # 6.25% of 32GB
 effective_cache_size = 24GB   # 75% of 32GB
wal_buffers = 64MB            # 适当大小的WAL缓冲区

# WAL配置
wal_level = replica
checkpoint_timeout = 30min
max_wal_size = 8GB
min_wal_size = 2GB
checkpoint_completion_target = 0.9
wal_compression = on

# 连接配置
max_connections = 500
listen_addresses = '*'
port = 5432

# 日志配置
log_min_duration_statement = 500ms
log_line_prefix = '%m [%p] %q%u@%d %a '
log_destination = 'stderr,csvlog'
logging_collector = on
log_directory = 'pg_log'
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0
log_autovacuum_min_duration = 500ms

# 自动清理配置
autovacuum = on
autovacuum_max_workers = 8
autovacuum_vacuum_scale_factor = 0.02
autovacuum_analyze_scale_factor = 0.01
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = -1

# 查询优化配置
random_page_cost = 1.1  # SSD存储
seq_page_cost = 1.0
cpu_tuple_cost = 0.01
cpu_index_tuple_cost = 0.005
cpu_operator_cost = 0.0025
```


Q1：如何查看当前配置？

```bash
-- 查看所有配置参数
SHOW ALL;

-- 查看特定参数
SHOW shared_buffers;

-- 从pg_settings表查询
SELECT name, setting, unit, short_desc FROM pg_settings WHERE name = 'shared_buffers';

-- 查看配置文件位置
SHOW config_file;

```

```bash
-- 动态修改参数
ALTER SYSTEM SET shared_buffers = '2GB';

-- 重新加载配置
SELECT pg_reload_conf();
```
