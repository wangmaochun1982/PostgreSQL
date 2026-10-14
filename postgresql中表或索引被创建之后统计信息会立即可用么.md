当我们创建表或者索引之后，他们的统计信息是自动可用么？更精确地描述一下：第一步，我们先创建或加载一个表；第二步，在表上创建一个索引。那我们是默认就有了可用的统计信息，或者还是需要等待一个autovacuum或手动analyze才可用呢？

我们测试一下吧。

首先禁用一下autovacuum,以免在后台自动执行。

```bash
postgres=# \! ps -ef|grep autov |grep -v grep
postgres  3741  3735  0 09:15 ?        00:00:00 postgres: autovacuum launcher  
postgres=# alter system set autovacuum=off;
ALTER SYSTEM
postgres=# select * from pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)
 
postgres=# \! ps -ef|grep autov |grep -v grep
postgres=#
```

创建用于测试的表：


```sql
postgres=# \! cat a.sql
drop table if exists t;
create table t
as select a.*, md5(a::varchar) from generate_series(1,5000000) a;
postgres=# \i a.sql
psql:a.sql:1: NOTICE:  table "t" does not exist, skipping
DROP TABLE
SELECT 5000000
```

创建一个索引：

```sql
postgres=# create index i1 on t(a);
CREATE INDEX
postgres=# \d+ t
                                     Table "public.t"
 Column |  Type   | Collation | Nullable | Default | Storage  | Stats target | Description
--------+---------+-----------+----------+---------+----------+--------------+-------------
 a      | integer |           |          |         | plain    |              |
 md5    | text    |           |          |         | extended |              |
Indexes:
    "i1" btree (a)
 
postgres=#

```

来看看是否已经有了统计信息。

```sql
postgres=# select stanullfrac,stawidth,stadistinct from pg_statistic where starelid = 't'::regclass;
 stanullfrac | stawidth | stadistinct
-------------+----------+-------------
(0 rows)
 
postgres=#
```

从结果看，目前表还没有统计信息。

那索引呢？

```sql
postgres=# select stanullfrac,stawidth,stadistinct from pg_statistic where starelid = 'i1'::regclass;
 stanullfrac | stawidth | stadistinct
-------------+----------+-------------
(0 rows)
 
postgres=#
```

也没有索引的统计信息！

现在我们手动analyze一下表：

```sql
postgres=# analyze t;
ANALYZE
postgres=# analyze i1;
WARNING:  skipping "i1" --- cannot analyze non-tables or special system tables
ANALYZE
postgres=#
```

很显然，我们不能对索引执行analyze操作。再来看看表和索引上是否有了统计信息：

```sql
postgres=# select stanullfrac,stawidth,stadistinct from pg_statistic where starelid = 't'::regclass;
 stanullfrac | stawidth | stadistinct
-------------+----------+-------------
           0 |        4 |          -1
           0 |       33 |          -1
(2 rows)
 
postgres=# select stanullfrac,stawidth,stadistinct from pg_statistic where starelid = 'i1'::regclass;
 stanullfrac | stawidth | stadistinct
-------------+----------+-------------
(0 rows)
 
postgres=#
```

表上已经有了统计信息，但是索引上没有统计信息。因为analyze只能对表执行，但是不能对索引执行。

对于常规的索引，不会在pg_statistic中内容，因为这样就与原表中的列的统计信息冗余了。但是，对于函数索引，是有自己的统计信息的。

```sql
postgres=# create index i2 on t(lower(a::text));
CREATE INDEX
postgres=# select stanullfrac,stawidth,stadistinct from pg_statistic where starelid = 'i2'::regclass;
 stanullfrac | stawidth | stadistinct
-------------+----------+-------------
(0 rows)
 
postgres=# analyze t;
ANALYZE
postgres=# select stanullfrac,stawidth,stadistinct from pg_statistic where starelid = 'i2'::regclass;
 stanullfrac | stawidth | stadistinct
-------------+----------+-------------
           0 |       10 |          -1
(1 row)
 
postgres=#
```

因此，当autovacuum被关闭后，如果我们不手动执行analyze，不会自动有可用的统计信息。

如果将autovacuum开启呢？

```sql
postgres=# alter system set autovacuum=on;
ALTER SYSTEM
postgres=# select * from pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)
postgres=# \i a.sql
DROP TABLE
SELECT 5000000
postgres=# select stanullfrac,stawidth,stadistinct from pg_statistic where starelid = 't'::regclass;
 stanullfrac | stawidth | stadistinct
-------------+----------+-------------
(0 rows)
```

创建完之后，就立即查询，我们可以看到，表还是没有统计信息。

等几秒之后，再次查看：

```sql
postgres=# select stanullfrac,stawidth,stadistinct from pg_statistic where starelid = 't'::regclass;
 stanullfrac | stawidth | stadistinct
-------------+----------+-------------
           0 |        4 |          -1
           0 |       33 |          -1
(2 rows)
 
postgres=#
```

统计信息已经有了。

结论：如果在加载表之后，需要当前的统计信息，你最好立即执行一个手动的analyze操作。否则，autovacuum会自动收集，不过需要等一会。

