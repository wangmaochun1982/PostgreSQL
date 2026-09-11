在 CentOS/RHEL 上安装

```bash
# 安装 Barman 仓库
yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# 安装 Barman
yum install -y barman

# 启动并启用 Barman 服务
systemctl start barman
systemctl enable barman
```

Barman 主配置

主配置文件（barman.conf）

```conf
# Barman 主配置文件
[barman]
; Barman 主目录
barman_home = /var/lib/barman
; Barman 用户
barman_user = barman
; 日志文件位置
log_file = /var/log/barman/barman.log
; 日志级别
log_level = INFO
; 压缩备份
compression = gzip
; 备份保留策略
retention_policy = RECOVERY WINDOW OF 14 DAYS
; 网络超时时间
network_compression = true
; 并行备份数量
parallel_jobs = 4
; 备份目录权限
backup_directory_permissions = 0700
```

配置 PostgreSQL 主库

1. 创建 Barman 用户

```bash
-- 在主库上创建 Barman 用户
CREATE ROLE barman WITH REPLICATION LOGIN PASSWORD 'strong_password';

-- 配置 pg_hba.conf
host replication barman 0.0.0.0/0 md5
host all barman 0.0.0.0/0 md5

-- 重新加载配置
SELECT pg_reload_conf();
```

2. 配置主库连接（main.conf）

```conf
; Barman 主库配置
[main]
; 主库主机地址
host = postgres-primary
; 主库端口
port = 5432
; 主库用户
user = barman
; 主库数据库名
dbname = postgres
; 备份方法
backup_method = postgres
; 备份选项
backup_options = exclusive_backup
; 流复制归档
treaming_archiver = on
; 复制槽名称
slot_name = barman_main
; 归档命令
archiver = on
; WAL 归档路径
archive_command = 'rsync -a %p barman@barman-server:/var/lib/barman/main/incoming/%f'
; 压缩备份
compression = gzip
; 并行备份
parallel_jobs = 4
; 备份保留策略
retention_policy = RECOVERY WINDOW OF 14 DAYS
```

Barman 基本使用
```bash
# 检查 Barman 主库连接状态
barman check main

# 查看主库详细信息
barman show-server main


```

执行全量备份

```bash
# 执行全量备份
barman backup main

# 查看备份列表
barman list-backups main

# 查看备份详情
barman show-backup main latest
```

执行增量备份

```bash
# 执行增量备份
barman backup main --incremental

# 查看增量备份详情
barman show-backup main latest
```

恢复备份
1. 基础恢复

```bash
# 恢复到指定目录
barman recover main latest /var/lib/postgresql/15/main

# 恢复到指定时间点
barman recover main latest /var/lib/postgresql/15/main --target-time "2024-01-24 12:00:00"

# 恢复到指定 WAL 位置
barman recover main latest /var/lib/postgresql/15/main --target-lsn "0/12345678"
```

2. 远程恢复


```bash
# 远程恢复到从库
barman recover main latest postgres@slave-server:/var/lib/postgresql/15/main

```

Barman 高级管理
备份保留与清理

```bash
# 查看保留策略
barman show-server main | grep retention_policy

# 执行保留策略
barman cron

# 手动清理旧备份
barman delete main oldest

# 清理特定备份
barman delete main 20240124T120000
```
