创建表空间

语法说明
创建表空间时，需要指定表空间名称、所有者和物理存储位置。所有者默认为当前用户，存储位置必须是一个已存在的目录，且 PostgreSQL 用户必须对该目录有读写权限。

```sql
-- 创建表空间语法
CREATE TABLESPACE tablespace_name
    OWNER user_name
    LOCATION '/path/to/tablespace';

```

以下是几个常见的表空间创建示例，根据不同的存储介质和使用场景进行规划：

```sql
-- 示例1：创建 SSD 存储的表空间
-- SSD 适合存储频繁访问的热数据，如核心业务表和索引
-- 优点：I/O 性能高，响应速度快
-- 缺点：成本较高，容量相对较小
CREATE TABLESPACE ssd_tablespace
    OWNER postgres
    LOCATION '/data/ssd/postgresql';

-- 示例2：创建普通 HDD 存储的表空间
-- HDD 适合存储访问频率较低的冷数据，如历史归档数据
-- 优点：成本低，容量大
-- 缺点：I/O 性能相对较低
CREATE TABLESPACE hdd_tablespace
    OWNER postgres
    LOCATION '/data/hdd/postgresql';

-- 示例3：创建带有自动分区的表空间
-- 通过符号链接或 LVM 实现存储空间的动态扩展
-- 适用于数据量增长较快的场景
CREATE TABLESPACE partitioned_tablespace
    OWNER postgres
    LOCATION '/data/partitions/ts_01';
```


<img width="792" height="366" alt="image" src="https://github.com/user-attachments/assets/e71193ef-e060-4a35-957c-4473ca39e6a3" />

表空间权限管理

```sql
-- 授予用户表空间访问权限
-- CREATE 权限允许用户在表空间中创建对象
GRANT CREATE ON TABLESPACE ssd_tablespace TO app_user;

-- 撤销用户表空间权限
-- 当用户不再需要使用表空间时，应及时回收权限
REVOKE CREATE ON TABLESPACE hdd_tablespace FROM app_user;

-- 查看表空间权限
-- 定期审计表空间权限，确保权限分配合理
SELECT 
    tablespace_name,  -- 表空间名称
    grantee,          -- 权限获得者
    privilege_type,   -- 权限类型
    is_grantable      -- 是否可转授
FROM information_schema.tablespace_privileges
WHERE tablespace_name = 'ssd_tablespace';
```

 

1. 单个表和索引迁移

```sql
-- 迁移单个表到指定表空间
-- 语法：ALTER TABLE 表名 SET TABLESPACE 目标表空间名
ALTER TABLE schema_name.table_name SET TABLESPACE tablespace_name;



-- 迁移表的索引到指定表空间
-- 语法：ALTER INDEX 索引名 SET TABLESPACE 目标表空间名
ALTER INDEX index_name SET TABLESPACE tablespace_name;
```

2. 表的所有索引迁移

```sql
-- 迁移表的所有索引到指定表空间
-- 使用匿名块批量处理
DO $$
DECLARE
    rec RECORD;  -- 用于存储查询结果的记录变量
BEGIN
    -- 遍历表的所有索引
    FOR rec IN 
        SELECT indexname 
        FROM pg_indexes 
        WHERE tablename = 'your_table'  -- 替换为实际表名
    LOOP
        -- 动态执行 ALTER INDEX 命令
        EXECUTE format('ALTER INDEX %I SET TABLESPACE ssd_tablespace', rec.indexname);
    END LOOP;
END;
$$;
```

```sql
-- 批量迁移指定表空间的所有表
-- 使用匿名块遍历并迁移所有表
DO $$
DECLARE
    rec RECORD;  -- 用于存储查询结果的记录变量
BEGIN
    -- 遍历原表空间中的所有表
    FOR rec IN 
        SELECT schemaname || '.' || tablename AS full_name
        FROM pg_tables
        WHERE tablespace = 'old_tablespace'::regnamespace  -- 替换为实际表空间名
    LOOP
        -- 动态执行 ALTER TABLE 命令
        EXECUTE format('ALTER TABLE %s SET TABLESPACE new_tablespace', rec.full_name);
        -- 输出迁移信息
        RAISE NOTICE '迁移表: %', rec.full_name;
    END LOOP;
END;
$$;
```

3.整个表空间迁移

```sql
-- 创建表空间迁移脚本
CREATE OR REPLACE FUNCTION generate_tablespace_migration_script(
    source_tablespace NAME,
    target_tablespace NAME
) RETURNS TEXT AS $$
DECLARE
    v_sql TEXT := '';
    rec RECORD;
BEGIN
    -- 生成表迁移脚本
    FOR rec IN 
        SELECT 'ALTER TABLE ' || schemaname || '.' || tablename || 
               ' SET TABLESPACE ' || target_tablespace || ';' AS stmt
        FROM pg_tables
        WHERE tablespace = source_tablespace::regnamespace
    LOOP
        v_sql := v_sql || rec.stmt || E'\n';
    END LOOP;
    
    -- 生成索引迁移脚本
    FOR rec IN 
        SELECT 'ALTER INDEX ' || schemaname || '.' || indexname || 
               ' SET TABLESPACE ' || target_tablespace || ';' AS stmt
        FROM pg_indexes
        WHERE tablespace = source_tablespace::regnamespace
    LOOP
        v_sql := v_sql || rec.stmt || E'\n';
    END LOOP;
    
    RETURN v_sql;
END;
$$ LANGUAGE plpgsql;

-- 生成迁移脚本
SELECT generate_tablespace_migration_script('old_tablespace', 'new_tablespace');
```

在线迁移注意事项

```sql
#!/bin/bash
# 表空间在线迁移脚本

set -e

SOURCE_TABLESPACE="old_tablespace"
TARGET_TABLESPACE="new_tablespace"
LOG_FILE="/var/log/pg_tablespace_migration.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a ${LOG_FILE}
}

# 迁移前检查
pre_migration_check() {
    log "执行迁移前检查..."
    
    # 检查目标表空间是否存在
    psql -c "SELECT 1 FROM pg_tablespace WHERE spcname = '${TARGET_TABLESPACE}';" || {
        log "错误: 目标表空间不存在"
        exit 1
    }
    
    # 检查源表空间使用情况
    psql -c "
        SELECT 
            '表: ' || schemaname || '.' || tablename AS object,
            pg_size_pretty(pg_relation_size(schemaname || '.' || tablename)) AS size
        FROM pg_tables
        WHERE tablespace = '${SOURCE_TABLESPACE}'::regnamespace
        ORDER BY pg_relation_size(schemaname || '.' || tablename) DESC
        LIMIT 10;
    "
}

# 执行迁移
execute_migration() {
    log "开始迁移表..."
    
    psql << EOF
DO \$\$
DECLARE
    rec RECORD;
BEGIN
    FOR rec IN 
        SELECT schemaname, tablename
        FROM pg_tables
        WHERE tablespace = '${SOURCE_TABLESPACE}'::regnamespace
    LOOP
        EXECUTE format('ALTER TABLE %I.%I SET TABLESPACE ${TARGET_TABLESPACE}', 
            rec.schemaname, rec.tablename);
        RAISE NOTICE '已迁移表: %.%', rec.schemaname, rec.tablename;
    END LOOP;
END;
\$\$;
EOF

    log "开始迁移索引..."
    
    psql << EOF
DO \$\$
DECLARE
    rec RECORD;
BEGIN
    FOR rec IN 
        SELECT schemaname, indexname
        FROM pg_indexes
        WHERE tablespace = '${SOURCE_TABLESPACE}'::regnamespace
    LOOP
        EXECUTE format('ALTER INDEX %I.%I SET TABLESPACE ${TARGET_TABLESPACE}', 
            rec.schemaname, rec.indexname);
        RAISE NOTICE '已迁移索引: %.%', rec.schemaname, rec.indexname;
    END LOOP;
END;
\$\$;
EOF
}

# 验证迁移结果
verify_migration() {
    log "验证迁移结果..."
    
    psql << EOF
-- 检查源表空间是否还有对象
SELECT 
    '源表空间剩余对象: ' || COUNT(*)::TEXT AS status
FROM (
    SELECT tablename FROM pg_tables WHERE tablespace = '${SOURCE_TABLESPACE}'::regnamespace
    UNION ALL
    SELECT indexname FROM pg_indexes WHERE tablespace = '${SOURCE_TABLESPACE}'::regnamespace
) objects;

-- 检查目标表空间对象数量
SELECT 
    '目标表空间对象数: ' || COUNT(*)::TEXT AS status
FROM (
    SELECT tablename FROM pg_tables WHERE tablespace = '${TARGET_TABLESPACE}'::regnamespace
    UNION ALL
    SELECT indexname FROM pg_indexes WHERE tablespace = '${TARGET_TABLESPACE}'::regnamespace
) objects;
EOF
}

# 主函数
pre_migration_check
execute_migration
verify_migration

log "表空间迁移完成"
```


# 表空间监控

```sql
-- 创建表空间监控视图
CREATE OR REPLACE VIEW pg_tablespace_monitor AS
SELECT 
    spcname AS tablespace_name,
    pg_tablespace_location(oid) AS location,
    pg_size_pretty(pg_tablespace_size(spcname)) AS total_size,
    pg_tablespace_size(spcname) AS total_bytes,
    CASE 
        WHEN spcname = 'pg_default' THEN '系统默认表空间'
        WHEN spcname = 'pg_global' THEN '系统全局表空间'
        ELSE '用户表空间'
    END AS tablespace_type,
    (SELECT COUNT(*) FROM pg_class WHERE reltablespace = oid) AS object_count,
    now() AS check_time
FROM pg_tablespace;

-- 表空间使用率告警视图
CREATE OR REPLACE VIEW pg_tablespace_usage_alert AS
SELECT 
    tablespace_name,
    location,
    total_size,
    object_count,
    CASE 
        WHEN total_bytes / 1024 / 1024 / 1024 > 100 THEN '🔴 超过 100GB'
        WHEN total_bytes / 1024 / 1024 / 1024 > 50 THEN '🟡 超过 50GB'
        ELSE '🟢 正常'
    END AS size_status
FROM pg_tablespace_monitor
WHERE tablespace_name NOT LIKE 'pg_%';

-- 详细表空间使用分析
SELECT 
    t.spcname AS tablespace,
    pg_tablespace_location(t.oid) AS path,
    pg_size_pretty(pg_tablespace_size(t.spcname)) AS size,
    COALESCE(SUM(pg_relation_size(c.oid)), 0) AS used_by_tables,
    COALESCE(SUM(pg_indexes_size(c.oid)), 0) AS used_by_indexes,
    pg_tablespace_size(t.spcname) - COALESCE(SUM(pg_relation_size(c.oid)), 0) - 
        COALESCE(SUM(pg_indexes_size(c.oid)), 0) AS overhead
FROM pg_tablespace t
LEFT JOIN pg_class c ON c.reltablespace = t.oid
WHERE t.spcname NOT LIKE 'pg_%'
GROUP BY t.spcname, t.oid
ORDER BY pg_tablespace_size(t.spcname) DESC;
```


增长趋势监控

```sql
-- 创建表空间历史记录表
CREATE TABLE IF NOT EXISTS pg_tablespace_history (
    id SERIAL PRIMARY KEY,
    tablespace_name VARCHAR(64) NOT NULL,
    size_bytes BIGINT NOT NULL,
    recorded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 创建定期收集任务
CREATE OR REPLACE PROCEDURE pg_collect_tablespace_metrics()
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO pg_tablespace_history (tablespace_name, size_bytes)
    SELECT 
        spcname,
        pg_tablespace_size(spcname)
    FROM pg_tablespace
    WHERE spcname NOT LIKE 'pg_%';
    
    -- 清理 30 天前的历史数据
    DELETE FROM pg_tablespace_history 
    WHERE recorded_at < now() - INTERVAL '30 days';
END;
$$;

-- 计算增长趋势
SELECT 
    tablespace_name,
    MIN(size_bytes) AS min_size,
    MAX(size_bytes) AS max_size,
    MAX(size_bytes) - MIN(size_bytes) AS growth,
    ROUND((MAX(size_bytes) - MIN(size_bytes))::numeric / 
        EXTRACT(EPOCH FROM (MAX(recorded_at) - MIN(recorded_at))) / 86400, 2) AS daily_growth_bytes
FROM pg_tablespace_history
WHERE recorded_at > now() - INTERVAL '7 days'
GROUP BY tablespace_name
ORDER BY daily_growth_bytes DESC;

```

自动监控脚本

```sql
#!/bin/bash
# PostgreSQL 表空间监控脚本
# 文件名: pg_tablespace_monitor.sh

set -e

LOG_FILE="/var/log/pg_inspection/tablespace_monitor.log"
ALERT_THRESHOLD=80
CRITICAL_THRESHOLD=90

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a ${LOG_FILE}
}

# 检查所有表空间使用情况
check_tablespaces() {
    log "检查表空间使用情况..."
    
    psql << 'EOF'
SELECT 
    spcname AS tablespace,
    pg_tablespace_location(oid) AS location,
    pg_size_pretty(pg_tablespace_size(spcname)) AS size,
    100 - (
        100 * pg_tablespace_size(spcname) / 
        (SELECT SUM(pg_tablespace_size(spcname)) FROM pg_tablespace WHERE spcname NOT LIKE 'pg_%')
    ) AS free_pct
FROM pg_tablespace
WHERE spcname NOT LIKE 'pg_%'
ORDER BY free_pct ASC;
EOF
}

# 表空间增长趋势
check_growth_trend() {
    log "分析表空间增长趋势..."
    
    psql << 'EOF'
WITH daily_metrics AS (
    SELECT 
        tablespace_name,
        DATE(recorded_at) AS record_date,
        MAX(size_bytes) AS daily_size
    FROM pg_tablespace_history
    WHERE recorded_at > now() - INTERVAL '7 days'
    GROUP BY tablespace_name, DATE(recorded_at)
)
SELECT 
    tablespace_name,
    MIN(daily_size) AS start_size,
    MAX(daily_size) AS end_size,
    MAX(daily_size) - MIN(daily_size) AS weekly_growth,
    ROUND((MAX(daily_size) - MIN(daily_size))::numeric / 7 / 1024 / 1024, 2) AS avg_daily_growth_mb,
    ROUND((MAX(daily_size) - MIN(daily_size))::numeric / MIN(daily_size) * 100, 2) AS growth_rate_pct
FROM daily_metrics
GROUP BY tablespace_name
ORDER BY avg_daily_growth_mb DESC
LIMIT 10;
EOF
}

# 生成表空间报告
generate_report() {
    local report_file="/var/log/pg_inspection/tablespace_report_$(date +%Y%m%d).log"
    
    {
        echo "PostgreSQL 表空间监控报告"
        echo "生成时间: $(date '+%Y-%m-%d %H:%M:%S')"
        echo ""
        echo "=== 表空间使用情况 ==="
        psql -c "SELECT * FROM pg_tablespace_monitor;" 2>/dev/null
        echo ""
        echo "=== 增长趋势 ==="
        psql -c "SELECT * FROM pg_tablespace_growth_trend;" 2>/dev/null
    } > ${report_file}
    
    log "报告已生成: ${report_file}"
}

# 主函数
check_tablespaces
check_growth_trend
generate_report
```

表空间性能优化

```sql
-- 识别 I/O 热点表空间
SELECT 
    t.spcname AS tablespace,
    pg_tablespace_location(t.oid) AS location,
    SUM(heap_blks_read) AS total_reads,
    SUM(heap_blks_hit) AS total_hits,
    ROUND(100.0 * SUM(heap_blks_hit) / 
        NULLIF(SUM(heap_blks_hit) + SUM(heap_blks_read), 0), 2) AS hit_ratio,
    SUM(idx_blks_read) AS idx_reads,
    SUM(idx_blks_hit) AS idx_hits
FROM pg_statio_user_tables s
JOIN pg_class c ON c.oid = s.relid
JOIN pg_tablespace t ON t.oid = c.reltablespace
GROUP BY t.spcname, t.oid
ORDER BY total_reads DESC;

-- 优化建议
SELECT 
    schemaname || '.' || tablename AS table_name,
    seq_scan,
    seq_tup_read,
    idx_scan,
    idx_tup_fetch,
    CASE 
        WHEN seq_scan > 1000 THEN '建议移动到 SSD 表空间'
        ELSE '当前表空间合适'
    END AS recommendation
FROM pg_stat_user_tables
ORDER BY seq_scan DESC
LIMIT 20;
```
分层存储策略

```sql
分层存储策略
-- 创建分层存储策略表
CREATE TABLE storage_tier_policy (
    id SERIAL PRIMARY KEY,
    tier_name VARCHAR(50) NOT NULL,
    storage_type VARCHAR(50) NOT NULL,  -- 'ssd', 'hdd', 'archive'
    max_size BIGINT,
    min_free_space BIGINT,
    rules JSONB,
    enabled BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 插入分层策略
INSERT INTO storage_tier_policy (tier_name, storage_type, max_size, min_free_space, rules) VALUES
('tier_ssd', 'ssd', 500000000000, 50000000000, 
 '{"tables": ["orders", "products", "users"], "indexes": ["*"]}'::jsonb),
('tier_hdd', 'hdd', 2000000000000, 200000000000,
 '{"tables": ["logs", "archives"], "indexes": ["logs_*"]}'::jsonb),
('tier_archive', 'archive', NULL, 100000000000,
 '{"tables": ["old_*"], "retention_days": 365}'::jsonb);

-- 创建自动分层存储过程
CREATE OR REPLACE PROCEDURE pg_storage_tier_optimization()
LANGUAGE plpgsql
AS $$
DECLARE
    rec RECORD;
    v_source_tablespace TEXT;
    v_target_tablespace TEXT;
BEGIN
    -- 将频繁访问的小表移动到 SSD
    FOR rec IN 
        SELECT 
            schemaname || '.' || tablename AS full_name,
            t.spcname AS current_tablespace
        FROM pg_stat_user_tables s
        JOIN pg_tablespace t ON t.oid = s.reltablespace
        WHERE t.spcname = 'hdd_tablespace'
          AND pg_relation_size(schemaname || '.' || tablename) < 1000000000
          AND idx_scan > 10000
    LOOP
        -- 检查目标表空间是否有足够空间
        SELECT spcname INTO v_target_tablespace
        FROM storage_tier_policy 
        WHERE storage_type = 'ssd' AND enabled
        LIMIT 1;
        
        IF v_target_tablespace IS NOT NULL THEN
            EXECUTE format('ALTER TABLE %s SET TABLESPACE %s', 
                rec.full_name, v_target_tablespace);
            RAISE NOTICE '已移动表 %s 到 %s', rec.full_name, v_target_tablespace;
        END IF;
    END LOOP;
END;
$$;
```

表空间故障处理

```sql
-- 故障 1: 表空间目录不可用
-- 症状: ERROR: could not open relation mapping file

-- 恢复步骤:
-- 1. 检查目录权限
-- 2. 修复文件系统
-- 3. 重启 PostgreSQL 服务

-- 故障 2: 表空间磁盘满
-- 症状: ERROR: could not extend relation

-- 应急处理:
-- 1. 清理临时文件
SELECT pg_size_pretty(SUM(size)) FROM pg_temp_files;

-- 2. 清理未使用的索引
SELECT indexrelname, pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;

-- 3. 清理旧数据
DELETE FROM your_table WHERE created_at < now() - INTERVAL '1 year';

-- 故障 3: 表空间损坏
-- 症状: ERROR: invalid page in block X of relation

-- 恢复步骤:
-- 1. 从备份恢复表空间
-- 2. 或使用 pg_repair 工具
```


表空间恢复脚本

```sql
#!/bin/bash
# PostgreSQL 表空间故障恢复脚本
# 文件名: pg_tablespace_recovery.sh

set -e

TABLESPACE_NAME=$1
RECOVERY_ACTION=$2

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

# 检查表空间状态
check_tablespace_status() {
    log "检查表空间 ${TABLESPACE_NAME} 状态..."
    
    psql -c "
        SELECT 
            spcname,
            pg_tablespace_location(oid) AS location,
            CASE 
                WHEN pg_tablespace_size(spcname) > 0 THEN '正常'
                ELSE '异常'
            END AS status
        FROM pg_tablespace
        WHERE spcname = '${TABLESPACE_NAME}';
    "
    
    # 检查目录是否存在
    local location=$(psql -t -A -c "SELECT pg_tablespace_location(
        (SELECT oid FROM pg_tablespace WHERE spcname = '${TABLESPACE_NAME}')
    )")
    
    if [ -d "${location}" ]; then
        log "表空间目录存在: ${location}"
        ls -la "${location}"
    else
        log "警告: 表空间目录不存在: ${location}"
    fi
}

# 修复表空间权限
fix_permissions() {
    log "修复表空间权限..."
    
    local location=$(psql -t -A -c "SELECT pg_tablespace_location(
        (SELECT oid FROM pg_tablespace WHERE spcname = '${TABLESPACE_NAME}')
    )")
    
    if [ -d "${location}" ]; then
        sudo chown -R postgres:postgres "${location}"
        sudo chmod 700 "${location}"
        log "权限已修复"
    fi
}

# 重建表空间
rebuild_tablespace() {
    log "重建表空间..."
    
    echo "此操作将重新创建表空间，请确保已备份所有数据"
    read -p "确认继续? (yes/no): " confirm
    
    if [ "${confirm}" = "yes" ]; then
        # 删除旧表空间
        psql -c "DROP TABLESPACE IF EXISTS ${TABLESPACE_NAME};"
        
        # 创建新表空间
        psql -c "CREATE TABLESPACE ${TABLESPACE_NAME} LOCATION '${location}';"
        
        log "表空间已重建"
    fi
}

# 主函数
case "${RECOVERY_ACTION}" in
    check)
        check_tablespace_status
        ;;
    fix)
        fix_permissions
        ;;
    rebuild)
        rebuild_tablespace
        ;;
    *)
        echo "用法: $0 {check|fix|rebuild}"
        ;;
esac
```
