这里需要关注的名词有：relation、fork、filenode、segment等。

传统方式上，数据库群集使用的配置文件和数据文件一起存储在群集的数据目录中，通常称为 PGDATA。PGDATA 的常用位置是 /var/lib/pgsql/data。


PGDATA 目录包含几个子目录和控制文件。以下是本人在debain上安装的测试环境示例：

```code
root@db:/var/lib/postgresql/17/main# ls -l
total 20
drwx------ 5 postgres postgres   33 Jan  7 11:28 base                #该目录包含了每个数据库的子目录
drwx------ 2 postgres postgres 4096 Jan  7 13:34 global              #包含集群范围的表
drwx------ 2 postgres postgres    6 Jan  7 11:28 pg_commit_ts        #该目录包含了事务提交的时间戳数据
drwx------ 2 postgres postgres    6 Jan  7 11:28 pg_dynshmem         #该目录包含动态共享内存子系统的文件
drwx------ 4 postgres postgres   68 Jan  7 11:33 pg_logical          #该目录包含用于逻辑解码的状态数据
drwx------ 4 postgres postgres   36 Jan  7 11:28 pg_multixact        #该目录包含多事务状态数据(用于共享行锁)
drwx------ 2 postgres postgres    6 Jan  7 11:28 pg_notify           #包含监听和通知的状态数据
drwx------ 2 postgres postgres    6 Jan  7 11:28 pg_replslot         #包含复制槽数据
drwx------ 2 postgres postgres    6 Jan  7 11:28 pg_serial           #已提交的序列化事务数据
drwx------ 2 postgres postgres    6 Jan  7 11:28 pg_snapshots        #包含导出快照
drwx------ 2 postgres postgres    6 Jan  7 11:28 pg_stat             #包含统计子系统永久文件的子目录
drwx------ 2 postgres postgres    6 Jan  7 11:28 pg_stat_tmp         #包含统计子系统临时文件的子目录
drwx------ 2 postgres postgres   18 Jan  7 11:28 pg_subtrans         #包含子事务状态数据
drwx------ 2 postgres postgres    6 Jan  7 11:28 pg_tblspc           #包含指向表空间的符号链接
drwx------ 2 postgres postgres    6 Jan  7 11:28 pg_twophase         #包含prepared事务的状态文件
-rw------- 1 postgres postgres    3 Jan  7 11:28 PG_VERSION          #该文件包含了pg的主版本号
drwx------ 4 postgres postgres   77 Jan  7 11:28 pg_wal              #wal文件
drwx------ 2 postgres postgres   18 Jan  7 11:28 pg_xact             #事务的提交状态信息
-rw------- 1 postgres postgres   88 Jan  7 11:28 postgresql.auto.conf#通过alter system配置的参数
-rw------- 1 postgres postgres  130 Jan  7 11:28 postmaster.opts     #启动pg时的命令行选项
-rw------- 1 postgres postgres  109 Jan  7 11:28 postmaster.pid      #pid文件
root@db:/var/lib/postgresql/17/main#
```

集群中的每个数据库在PGDATA/base目录下，都有一个子目录，以数据库的oid命名。是数据库文件的默认存储位置。例如，我的测试环境中有三个库：

```bash
postgres=# select oid,datname from pg_database;
 oid |  datname 
-----+-----------
   5 | postgres
   1 | template1
   4 | template0
(3 rows)
 
postgres=#
```

在PGDATA/base目录下，有三个子目录：

```bash
postgres@db:~/17/main/base$ ls -l
total 36
drwx------ 2 postgres postgres 8192 Jan  7 11:31 1
drwx------ 2 postgres postgres 8192 Jan  7 11:28 4
drwx------ 2 postgres postgres 8192 Jan  7 13:34 5
 

```


每个表和索引都存储在一个单独的文件中。


对于普通的关系(relations)，这些文件以表或索引的文件节点号命名，文件节点号可以在 pg_class.relfilenode 中找到。

但对于临时关系，文件名的形式是 tBBB_FFF，其中 BBB 是创建文件的后端进程号，FFF 是文件节点号。

无论哪种情况，除了主文件（又称主分叉:main fork），每个表和索引都有一个空闲空间映射，其中存储了关系中可用的空闲空间信息。

空闲空间映射存储在一个文件中，文件名是文件节点编号加上后缀 _fsm。


比如创建一张表，然后插入和删除一些数据，观察磁盘上的文件:

```bash

postgres=# \c abce
You are now connected to database "abce" as user "postgres".

abce=# create table test(id int, age int);
CREATE TABLE
abce=# SELECT oid,relname,relfilenode,relkind FROM pg_class WHERE relname = 'test' AND relkind = 'r';
  oid  | relname | relfilenode | relkind
-------+---------+-------------+---------
 16390 | test    |       16390 | r
(1 row)
 
abce=# create index idx_id on test(id);
CREATE INDEX
abce=# SELECT oid,relname,relfilenode,relkind FROM pg_class WHERE relname = 'idx_id' AND relkind = 'i';
  oid  | relname | relfilenode | relkind
-------+---------+-------------+---------
 16393 | idx_id  |       16393 | i
(1 row)

```


以下是磁盘上的文件：

```bash

postgres@db:~/17/main/base/16389$ ls -l |grep 16390
-rw------- 1 postgres postgres 1073741824 Jan  9 13:33 16390
-rw------- 1 postgres postgres 1073741824 Jan  9 13:55 16390.1
-rw------- 1 postgres postgres 1073741824 Jan  9 13:56 16390.2
-rw------- 1 postgres postgres 1035739136 Jan  9 13:57 16390.3
-rw------- 1 postgres postgres    1064960 Jan  9 13:54 16390_fsm
-rw------- 1 postgres postgres     131072 Jan  9 13:54 16390_vm


postgres@db:~/17/main/base/16389$ ls -l |grep 16393
-rw------- 1 postgres postgres  870866944 Jan  9 13:54 16393
-rw------- 1 postgres postgres      98304 Jan  9 13:54 16393_fsm

```

表还有一个可见性映射，存储在一个后缀为 _vm 的分叉文件中，用于跟踪哪些页面没有死元组。

Unlogged 的表和索引有第三个分叉，称为初始化分叉，存储在后缀为 _init 的分叉中。

注意：

# 请注意，虽然表的文件节点(filenode)通常与其 OID 一致，但情况并不一定如此；某些操作，如 TRUNCATE、REINDEX、CLUSTER 和某些形式的 ALTER TABLE，可以在保留 OID 的同时更改文件节点。

此外，对于包括 pg_class 本身在内的某些系统目录，pg_class.relfilenode 包含 0。

这些目录的实际 filenode 编号存储在一个较低级别的数据结构中，可以使用 pg_relation_filenode() 函数获取。


# 当表或索引超过 1GB 时，会被划分为GB大小的段(segments)。

第一个段的文件名与 filenode 相同；随后的段分别命名为 filenode.1、filenode.2 等。

这种安排可以避免在有文件大小限制的平台上出现问题。(实际上，1 GB 只是默认的段大小。在构建 PostgreSQL 时，可以使用配置选项 --with-segsize 来调整段的大小）。

原则上，空闲空间映射和可见性映射分叉也可能需要多个分段，但这在实践中不太可能发生。

# 如果表中的列有很大的条目，那么它就会有一个相关的 TOAST 表，这个表用于行外存储字段值，因为字段值太大，无法保存在表的行中。

pg_class.reltoastrelid 将表链接到它的 TOAST 表（如果有的话）。


# 表空间使情况变得更加复杂。

用户定义的每个表空间在 PGDATA/pg_tblspc 目录中都有一个符号链接，指向物理表空间目录。

该符号链接以表空间的 OID 命名。

在物理表空间目录下有一个子目录，其名称取决于 PostgreSQL 服务器版本，如 PG_9.0_201008051。(使用该子目录的原因是，数据库的连续版本可以使用相同的 CREATE TABLESPACE 位置值，而不会发生冲突）。

在特定于版本的子目录中，每个数据库都有一个表空间元素子目录，以数据库的 OID 命名。

表和索引使用文件节点命名方案存储在该目录中。


# pg_default 表空间不能通过 pg_tblspc 访问，而是对应于 PGDATA/base。同样，pg_global 表空间也不是通过 pg_tblspc 访问的，而是对应于 PGDATA/global。


# pg_relation_filepath() 函数显示任何关系的整个路径（相对于 PGDATA）。

它通常可以代替记忆上述许多规则。但要注意的是，该函数只给出关系主分叉第一个段的名称--你可能需要附加段号和/或 _fsm、_vm 或 _init 才能找到与关系相关的所有文件。
 

 临时文件创建在 PGDATA/base/pgsql_tmp 目录中，如果为临时文件指定的表空间不是 pg_default，则创建于表空间目录的 pgsql_tmp 子目录中。临时文件的名称格式为 pgsql_tmpPPP.NNN，其中 PPP 是拥有该文件的后端 PID，NNN 则用于区分该后端不同的临时文件。
