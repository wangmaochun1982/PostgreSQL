在 PostgreSQL 的 postgresql.conf 中，控制 WAL 归档的关键参数如下：

```conf

# 1. 开启归档模式（必须重启数据库服务生效）
archive_mode = on

# 2. 定义归档命令（把已满的 WAL 文件复制到备份存储路径）
# %p 代表待归档 WAL 文件的绝对路径，%f 代表 WAL 文件名
#archive_command = 'test ! -f /backup/wal_archive/%f && cp %p /backup/wal_archive/%f'
archive_command = 'cp %p /backup/postgresql/wal/%f'             # command to use to archive a WAL file

# 3. 超时强制归档（单位：秒）。即使 WAL 未写满（16MB），超过指定时间也会切日志并归档
archive_timeout = 300

# 4. 保留的最高 WAL 级别（要支持归档或从库同步，需设置为 replica 或 logical）
wal_level = replica

```

注意： archive_command 必须保证原子性和幂等性（返回值为 0 表示成功）。生产环境中建议配合脚本或备份工具使用，避免磁盘写满导致数据库停服务。


WAL 文件的生命周期通常包含 生成 -> 传输归档 -> 全量基线关联 -> 自动清理/压缩。

1. 归档日志清理策略WAL 归档文件绝对不能单独直接手动清理，必须结合全量物理备份（Base Backup）进行。
  
   保留原则：如果要恢复到时刻T，必须拥有 T 之前的最后一次全量基线备份 + 该基线备份开始之后产生的所有 WAL 归档文件。

   清理规则：只有比“最老需要的全量备份”还要早的 WAL 归档文件才能安全删除。

2. 手动/脚本清理工具：pg_archivecleanup

   ```bash
# 查看帮助与模拟删除（-n 仅打印不实际删除）
pg_archivecleanup -n /backup/postgresql/wal/ 000000010000000200000010

# 实际清理：删除 000000010000000200000010 之前的所有 WAL 文件
pg_archivecleanup /backup/postgresql/wal/ 000000010000000200000010
   ```



