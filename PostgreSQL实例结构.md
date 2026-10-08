内容1：PostgreSQL实例简介

内容2：PostgreSQL初始化参数文件

内容3：PostgreSQL内存结构

内容4：PostgreSQL后台进程

1.1、PG实例简介

PostgreSQL实例：用来访问PostgreSQL数据库

一个实例对应一个数据库集簇

由内存和后台进程组成

<img width="758" height="296" alt="image" src="https://github.com/user-attachments/assets/54b0993b-5de9-4e11-94f5-1f7e0b0cc293" />

1.2、建立一个连接和创建一个会话

连接到一个PostgreSQL实例建立一个用户连接

创建一个会话

<img width="806" height="355" alt="image" src="https://github.com/user-attachments/assets/32f30b4a-2d22-43e2-851f-7055e48f061f" />


2、初始化参数文件

<img width="893" height="463" alt="image" src="https://github.com/user-attachments/assets/76d84924-46d7-4999-87c4-e857e123633b" />

两种类型的参数显式：在文件中对参数进行设置

隐式：文件中没有设置，使用Postgres默认值

可以存在多个初始化参数文件静态参数文件：postgresql.conf

动态参数文件：postgresql.auto.conf

可选参数文件：postgresql.conf.user

读取顺序postgresql.conf

postgresql.auto.conf

postgresql.conf.user


2.1、postgresql.conf参数文件

postgresql.conf参数文件文本文件

使用操作系统编辑器修改

手动修改

更改将在下次启动时生效

仅在实例启动期间读取

默认位置为$PGDATA


2.2、postgresql.auto.conf参数文件

postgresql.auto.conf参数文件文本文件

由Postgres服务器维护

支持用文本编辑器修改(不推荐)

ALTER SYSTEM命令修改的参数保存在该文件

能够在关闭和启动期间持续进行更改

可以实现自我调整参数值

默认位置为$PGDATA

改变一个参数的值，会在文件中自动添加参数

ALTER SYSTEM SET archive_mode = on;

恢复一个参数默认值，会在文件中自动删除参数

ALTER SYSTEM RESET archive_mode;


2.3、参数生效条件

参数生效条件 sighup：表示需要超级管理员修改，reload就能够生效。

superuser：表示使用超级管理员可以为普通用户、数据库、或者超级管理员自己修改。(注：有些参数是可以针对用户、数据库、实例)

postmaster：表示需要超级管理员修改，需要重启才能够生效。

user：表示普通用户可以修改该参数值，立即生效。

<img width="720" height="294" alt="image" src="https://github.com/user-attachments/assets/795183f9-430e-4c90-a85f-03200665fc82" />


3、Memory Architecture

PostgreSQL中的内存架构可以分为两大类：Local memory area –由每个后端进程分配给自己使用

Shared memory area –由PostgreSQL服务器的所有进程使用

<img width="690" height="319" alt="image" src="https://github.com/user-attachments/assets/ac4127c2-6d87-4bfc-896f-bee5b283d393" />


3.1、Memory Architecture

Local Memory Area：

<img width="949" height="262" alt="image" src="https://github.com/user-attachments/assets/bca0f5bc-214e-460f-83e3-11f7c5007919" />


Shared Memory Area：

<img width="949" height="299" alt="image" src="https://github.com/user-attachments/assets/7dbf5210-59c8-4efe-bf91-2655fb722337" />

4、Process Architecture

PostgreSQL进程结构：

postgres server process ：postgres服务器进程是与数据库群集管理相关的所有进程的父进程。

backend process：每个后端进程处理连接的客户端发出的所有查询和语句。

background processes：各种后台进程执行用于数据库管理的每个特性的进程(例如清空和检查点进程)。

replication associated processes：在与复制相关联的进程中，它们执行流式复制。

background worker process-：在版本9.3支持的后台工作进程中，它可以执行用户实现的任何处理。

PostgreSQL进程结构：

<img width="837" height="399" alt="image" src="https://github.com/user-attachments/assets/bff077e4-2edf-4045-ab17-9fb3c07749a6" />

4.1、Postgres Server Process

Postgres Server Processpostgres服务器进程是PostgreSQL服务器中所有进程的父进程。早期版本它被称为“postmaster”。

pg_ctl实用程序启动该进程，然后派生出各个后台进程。同时分配共享内存区域。

一个postgres服务器进程侦听一个网络端口，默认端口为5432。

4.2、Backend Processes

Backend Processes后端进程(也称为postgres)由postgres服务器进程启动，并处理一个连接的客户端发出的所有查询。

PostgreSQL允许多个客户端同时连接;配置参数max_connections控制客户端的最大数量。
<img width="657" height="310" alt="image" src="https://github.com/user-attachments/assets/a7007b41-a02e-49fc-aba0-1c97be0980e1" />


Background Processes

<img width="950" height="378" alt="image" src="https://github.com/user-attachments/assets/7eff3673-7255-419f-b57e-1236058998e1" />

postgres> pstree -p 9687-+= 00001 root /sbin/launchd\-+- 09687 postgres /usr/local/pgsql/bin/postgres -D /usr/local/pgsql/data |--= 09688 postgres postgres: logger process|--= 09690 postgres postgres: checkpointer process |--= 09691 postgres postgres: writer process|--= 09692 postgres postgres: wal writer process|--= 09693 postgres postgres: autovacuum launcher process |--= 09694 postgres postgres: archiver process|--= 09695 postgres postgres: stats collector process|--= 09697 postgres postgres: postgres sampledb 192.168.1.100(54924) idle\--= 09717 postgres postgres: postgres sampledb 192.168.1.100(54964) idle in transaction

