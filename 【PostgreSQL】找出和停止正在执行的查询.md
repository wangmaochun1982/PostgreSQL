1.找出对应的pid
 ```sql
SELECT pid, state, backend_start, substr(query, 0, 100) q
FROM pg_stat_activity
WHERE backend_type = 'client backend'
ORDER BY backend_start;

```
substr()函数调用避免sql太长，占据了大量屏幕；

backend_type='client backend',过滤掉background进程，比如autovacuum进程。

2.终止或取消进程
方式1：

 
```sql
SELECT pg_terminate_backend(pid);
```
pg_terminate_backend()彻底停止进程，导致连接关闭。事务会回滚，释放持有的锁。

pg_terminate_backend()隐含着immediately，而pg_cancel_backend()可能会延迟一段时间。

方式2：

 
```sql
SELECT pg_cancel_backend(pid);

```
pg_cancel_backend()只是中断正在运行的查询，连接仍然存在。当前事务或者检查点被中断。因此，如果连接使用了savepoint，仍然可以保持事务是开着的。

**查看阻塞某个特定进程的会话**

如果正在执行ALTER TABLE并发现它被阻塞，正在等待表锁，你可能希望终止持有该表锁的连接。从而允许ALTER TABLE继续进行。

可以通过查询pg_stat_activity找到阻塞ALTER TABLE的pid，如下所示：

```sql
SELECT pid, state, backend_start, substr(query, 0, 100) q
FROM pg_stat_activity
WHERE backend_type = 'client backend' AND query LIKE 'ALTER TABLE%'
ORDER BY backend_start;
```

找到alter table会话对应的pid，再通过函数pg_blocking_pids()找出阻塞alter table的哪些会话pid：(这里的blockdpid是上面alter table会话的pid)

```sql
SELECT 'SELECT pg_terminate_backend(' || pid || ');'
FROM pg_stat_activity
WHERE pid = ANY (pg_blocking_pids(blockedpid));
```

**查看和杀掉某个表上的会话…**

```sql
SELECT 'SELECT pg_terminate_backend(' || pid || ');'
FROM pg_stat_activity
WHERE query LIKE '%表名%'
  AND pid != pg_backend_pid();--排除当前的会话
```


**查看和杀掉时间超过N秒的会话**
例如10秒


```sql
SELECT 'SELECT pg_terminate_backend(' || pid || ');'
FROM pg_stat_activity
WHERE backend_type = 'client backend'
      AND pid != pg_backend_pid()
      AND backend_start < NOW() - '10 seconds'::interval;
```

**找出阻塞和被阻塞的会话**

```sql
SELECT
    activity.pid,
    activity.usename,
    activity.query,
    blocking.pid AS blocking_id,
    blocking.query AS blocking_query
FROM pg_stat_activity AS activity
JOIN pg_stat_activity AS blocking ON blocking.pid = ANY(pg_blocking_pids(activity.pid));
```
