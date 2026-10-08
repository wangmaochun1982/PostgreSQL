* 1.1、数据库集簇逻辑结构

数据库集簇逻辑结构 ：

<img width="964" height="393" alt="image" src="https://github.com/user-attachments/assets/4e06cda8-388a-4391-8298-238ca01b7341" />


1.2、对象标识符

PostgreSQL中的所有数据库对象都由各自的对象标识符(oid)在内部管理。

这些对象标识符是无符号的4字节整数。数据库对象和相应的oid之间的关系存储在适当的系统目录中，具体取决于对象的类型。

数据库的oid存储在pg_database中。

数据库的oid与对应的数据库目录名是一致的

2、数据库集簇物理结构

<img width="925" height="450" alt="image" src="https://github.com/user-attachments/assets/39f7fa90-e5c5-42a6-aae7-7469f4167365" />

每个数据库是base子目录下的子目录;

数据库目录名与相应的oid相同。

例如，当数据库sampledb的OID为16384时，其子目录名为16384。

```bash
cd $PGDATA

ls -ld base/16384

drwx------ 213 postgres postgres 7242 8 26 16:33 16384
```

2.1、其它目录结构

数据库集群的布局的主要文件和子目录如下所示：

<img width="977" height="377" alt="image" src="https://github.com/user-attachments/assets/ac8f6169-7750-4441-bde1-af623b27e3cc" />


<img width="960" height="485" alt="image" src="https://github.com/user-attachments/assets/a6a32254-4376-4a39-8f86-d722a94e6bd2" />


<img width="959" height="477" alt="image" src="https://github.com/user-attachments/assets/facb7dcf-7773-4317-923f-4f4a469de08c" />


3、Tablespaces

PostgreSQL Tablespaces

PostgreSQL中的表空间是基本目录之外的附加数据区域，此功能已在版本8.0中实现。

初始化数据库后默认的表空间有pg_default、pg_global。

pg_global表空间的物理文件位置在数据目录的global目录中，它用来保存系统表。

pg_default表空间的物理文件位置在数据目录的base子目录中，是template0和template1数据库的默认表空间。

创建数据库时，默认从template1数据库进行克隆，因此除非特别指定了新建数据库的表空间，否则默认使用template1使用的表空间，即pg_default表空间。


# PostgreSQL表空间物理文件位置

<img width="705" height="398" alt="image" src="https://github.com/user-attachments/assets/cdcdb102-c2fb-4827-a3de-c58e034242d7" />


创建表空间时产生的目录命名规则PG _ 'Major version' _ 'Catalogue version number'

例如：

```bash
sampledb=# create tablespace new_tblspc location '/home/postgres/tblspc';

$ ls -l /home/postgres/tblspc/ total 4

drwx------ 4 postgres postgres PG_12_201909212
```

新建表空间的目录由pg_tblspc子目录中的软链接寻址，链接名与表空间的OID值相同

<img width="956" height="394" alt="image" src="https://github.com/user-attachments/assets/390971ac-d739-4d58-8e11-b452c54164bf" />


如果在表空间下创建一个新的数据库(OID是90209)，那么它的目录将在版本特定的子目录下创建

```bash

ls -l /home/postgres/tblspc/PG_12_201909212 total 4

drwxr-x---. 2 postgres postgres 4096 Mar 30 09:27 90209

```

在base目录下创建的数据库上创建新表指定到新建的表空间

```bash
testdb=# create table test1 (id int) tablespace new_tblspc;

testdb=# SELECT pg_relation_filepath('test1'); pg_relation_filepath
```
---------------------------------------------

pg_tblspc/90208/PG_12_201909212/16385/90210 #在新表空间目录下创建数据库目录
