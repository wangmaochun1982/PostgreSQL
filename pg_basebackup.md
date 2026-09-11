<h2>pg_basebackup 配置</h2>

主库配置

```sql
-- 启用流复制
ALTER SYSTEM SET wal_level = 'replica';
ALTER SYSTEM SET max_wal_senders = '10';
ALTER SYSTEM SET wal_keep_size = '1GB';
ALTER SYSTEM SET hot_standby = 'on';

-- 创建复制用户
CREATE ROLE replication_user WITH REPLICATION LOGIN PASSWORD 'strong_password';

-- 配置 pg_hba.conf
host replication replication_user 0.0.0.0/0 md5

-- 重新加载配置
SELECT pg_reload_conf();
```


从库配置

```sql
-- 配置从库参数
ALTER SYSTEM SET hot_standby = 'on';
ALTER SYSTEM SET max_standby_streaming_delay = '30s';
ALTER SYSTEM SET wal_receiver_status_interval = '10s';
ALTER SYSTEM SET hot_standby_feedback = 'on';

-- 重新加载配置
SELECT pg_reload_conf();
```

# pg_basebackup 使用方法
```bash
# 执行全量备份
pg_basebackup -h localhost -U replication_user -D /backup/postgresql/full -F p -X stream -R -P

pg_basebackup -h localhost -p 5432 -U root -D /backup/postgresql/full/ -F tar -X stream -R -P

# 参数说明：
# -h：主库主机地址
# -U：复制用户名
# -D：备份目录
# -F p：使用纯文本格式
# -X stream：流式传输 WAL 文件
# -R：自动生成恢复配置
# -P：显示备份进度
```

压缩备份

```bash
# 使用 gzip 压缩
pg_basebackup -h localhost -U replication_user -D /backup/postgresql/full -F p -X stream -R -P -z

# 使用自定义压缩级别
pg_basebackup -h localhost -U replication_user -D /backup/postgresql/full -F p -X stream -R -P -Z 9

# 备份为 tar 格式并压缩
gpg_basebackup -h localhost -U replication_user -F t -X stream -R -P -z -D - > /backup/postgresql/full_backup.tar.gz
```

并行备份

```bash
# 使用 4 个并行进程
pg_basebackup -h localhost -U replication_user -D /backup/postgresql/full -F p -X stream -R -P -j 4

# 并行备份并压缩
pg_basebackup -h localhost -U replication_user -D /backup/postgresql/full -F p -X stream -R -P -j 4 -Z 5
```

增量备份准备

```bash
# 创建基础备份
pg_basebackup -h localhost -U replication_user -D /backup/postgresql/base -F p -X stream -R -P

# 记录备份的 WAL 位置
cat /backup/postgresql/base/backup_label
```

<h2>pg_basebackup 恢复方法</h2>

从全量备份恢复

```sql
# 停止 PostgreSQL
systemctl stop postgresql-15

# 清理数据目录
rm -rf /var/lib/postgresql/15/main/*

# 从备份恢复
rsync -avz /backup/postgresql/full/ /var/lib/postgresql/15/main/

# 启动 PostgreSQL
systemctl start postgresql-15
```

从 tar 备份恢复

```bash
# 停止 PostgreSQL
systemctl stop postgresql-15

# 清理数据目录
rm -rf /var/lib/postgresql/15/main/*

# 解压备份
mkdir -p /tmp/backup
tar -xf /backup/postgresql/full_backup.tar -C /tmp/backup
rsync -avz /tmp/backup/ /var/lib/postgresql/15/main/

# 启动 PostgreSQL
systemctl start postgresql-15
```

基于时间点的恢复

```bash
基于时间点的恢复
# 配置 recovery.conf
cat > /var/lib/postgresql/15/main/recovery.conf <<EOF
restore_command = 'cp /backup/postgresql/wal/%f %p'
recovery_target_time = '2024-01-24 12:00:00'
recovery_target_inclusive = true
EOF

# 启动 PostgreSQL
systemctl start postgresql-15
```
