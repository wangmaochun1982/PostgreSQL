PostgreSQL的内存结构
PostgreSQL的内存主要分成两类：

1.本地内存区域：每个后端进程(backend process)自己使用的

2.共享内存区域：所有进程共同使用

<img width="659" height="556" alt="image" src="https://github.com/user-attachments/assets/e82fb00f-cb87-46e1-844e-57c81d979fcc" />

## 本地内存区域,类似ORACLE的PGA
在postgresql中，每个后端进程(backend process)分配一片本地内存用于处理查询处理；

本地内存又分成子内存区域

# (1)work_mem

执行器使用work_mem区域用于对元组的排序，比如order by、distinct操作。也用于表的join操作，比如merge-join、hash-join操作

有参数work_mem控制，默认是4MB。

参数log_temp_files会控制记录临时文件名和大小。可以为排序、散列和临时查询结果创建临时文件。如果通过此设置启用，则在删除每个临时文件时将为其发出一个日志条目。当值为0时，记录所有临时文件信息;当值为正时，只记录大小大于或等于指定数据量的文件。如果指定此值时没有使用单位，则将其作为kb。默认设置是-1，它禁用这样的日志记录。只有超级用户才能更改此设置。

设置建议：

--从较小的值开始

--然后看看日志中的“temporary file”相关信息

--设置成temp文件的2-3倍

# (2)maintenance_work_mem

用于一些维护操作。比如vacuum、reindex、alter table add foreign key

由参数maintenance _work_mem控制大小。默认是64MB。由于数据库会话一次只能执行其中的一个操作，并且PostgreSQL不会同时运行许多操作，因此可以将maintenance_work_mem的值设置为明显大于work_mem的值。

设置建议：

--RAM*10%，最大不超过1GB

--如果vacuum有问题，可以设置的再大一点。

# (3)temp_buffers

存放临时表。temp_buffer参数控制。默认是8MB。

该参数设置每个数据库会话使用的临时缓冲区的最大数量。会话本地缓冲区仅用于访问临时表。可以在单个会话中更改此参数的设置，但只能在会话中首次使用临时表之前更改。

PostgreSQL利用这个内存区域来保存每个会话的临时表，当连接关闭时，这些临时表将被清除。

## 共享内存区域  类似ORACLE的SGA
# (1)shared buffer

PostgreSQL将表和索引中的页面从持久存储加载到共享缓冲池中，然后直接对它们进行操作。

由参数shared_buffers控制，默认是128MB。

# (2)wal buffer

存放wal数据

(3)commit log

提交日志(CLOG)保存所有事务的状态，是并发控制机制的一部分。提交日志分配共享内存，并在整个事务处理过程中使用。

事务有以下四种状态：

in_progress、commited、aborted、sub-commited

 
其他参数Effective_cache_size。详见postgresql的effective_cache_size



# PostgreSQL 内存主要分为两大类：

本地存储空间：由每个后端进程分配供自己使用。
共享存储空间：被 PostgreSQL 服务中所有的进程所共享。

<img width="660" height="500" alt="image" src="https://github.com/user-attachments/assets/ad4cb158-a106-4bb7-beb6-a0054a58c7fa" />


# PostgreSQL 内存调优

# Shared_buffers：此参数指定用于共享内存缓冲区的内存量。shared_buffers 参数决定为服务器缓存数据专用多少内存，相当于 Oracle 数据库中的 SGA。

```code
设置 shared_buffers 的建议

在 2 GB 内存以下，将 shared_buffers 的值设置为系统总内存的 20%。
在 32 GB 内存以下，将 shared_buffers 的值设置为系统总内存的 25%。
32 GB 以上内存，将 shared_buffers 的值设置为 8 GB。

建议：
配置建议：通常设为物理内存的 25%

注意事项：如果服务器内存非常大（如 64GB+），可以适当调高到 30%~40%，但由于 OS 层面还有文件系统缓存（Page Cache），盲目开得过大反而可能导致双重缓存降低效率。

```

# 设置 work_mem 的建议

```code

如果有并发的 M 个进程，每个进程有 N 个 HASH 操作，则需要分配的内存是 M * N * work_mem。所以不要把这个设置得太大，容易 OOM。

默认值为 work_mem = 4 MB。

设置 work_mem 的建议
从低值开始：32-64 MB。
在日志中寻找"临时文件"行，设置为最大临时文件的 2-3 倍。

配置建议：

对于高并发系统，通常设为 4MB ~ 64MB。

如果系统并发较低但 OLAP 分析查询非常多、涉及大量排序，可以针对特定会话或用户临时调大（SET work_mem = '256MB';），而不是全局盲目改大。

监控指标：通过日志或统计视图检查是否有大量 temp file 产生。如果频繁出现磁盘临时文件，说明当前 work_mem 偏小。
```

# maintenance_work_mem


```code
设置 maintenance_work_mem 的建议

设置系统内存的 10%，最大 1 GB。
如果你有真空问题，也许你可以把它设置得更高。
```
