postgresql中默认会有三个数据库：postgres、template0、template1。

```bash
postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges  
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =T/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(3 rows)
 
postgres=#
```

客户端默认会连接到postgres库。可以删除该库，不过会影响默认客户端连接。

删除了postgres库之后，可以借助模板库template1再创建postgres库：

```bash

$ psql template1
psql (11.9)
Type "help" for help.
 
template1=# drop database postgres;
DROP DATABASE
template1=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges  
-----------+----------+----------+-------------+-------------+-----------------------
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(2 rows)
 
template1=# create database postgres;
CREATE DATABASE
template1=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges  
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(3 rows)
 
template1=#
```


其实，在使用create database db_name语句创建新库的时候，就是创建模板库template1的一个拷贝。

那如果我修改了template1库会怎样呢？

```sql
$ psql template1
psql (11.9)
Type "help" for help.
 
template1=# create table my_test_tab(a int);
CREATE TABLE
template1=# create extension hstore;
CREATE EXTENSION
template1=# \dx
                           List of installed extensions
  Name   | Version |   Schema   |                   Description                   
---------+---------+------------+--------------------------------------------------
 hstore  | 1.5     | public     | data type for storing sets of (key, value) pairs
 plpgsql | 1.0     | pg_catalog | PL/pgSQL procedural language
(2 rows)
 
template1=#

```

修改以后，再创建新库的时候，新库也会包含上面的表和扩展：

```sql
template1=# create database db_test;
CREATE DATABASE
template1=# \c db_test
You are now connected to database "db_test" as user "postgres".
db_test=# \dx
                           List of installed extensions
  Name   | Version |   Schema   |                   Description                   
---------+---------+------------+--------------------------------------------------
 hstore  | 1.5     | public     | data type for storing sets of (key, value) pairs
 plpgsql | 1.0     | pg_catalog | PL/pgSQL procedural language
(2 rows)
 
db_test=# \d
            List of relations
 Schema |    Name     | Type  |  Owner  
--------+-------------+-------+----------
 public | my_test_tab | table | postgres
(1 row)
 
db_test=# 
```

无论，在template1中加入了什么，都会在之后新建的库中。

那template0的用途是什么呢？

```bash
db_test=# select datname,datallowconn,datistemplate from pg_database order by 3;
  datname  | datallowconn | datistemplate
-----------+--------------+---------------
 postgres  | t            | f
 db_test   | t            | f
 template1 | t            | t
 template0 | f            | t
(4 rows)
 
db_test=#
```

从这里可以看到，只有template0库对应的datallowconn字段的值是F。

这就是上面重建postgres的时候先登录template1而不是template0的原因。

template0是默认的不可修改的数据库。

不建议用户对template0做任何修改。

在初始化后的空实例中，template0和template1是完全相同的。

为什么需要两个模板库呢？假设你搞乱了template1，还可以通过template0恢复template1。

如果你想创建自己的模板库，只需将你选中库对应的datistemplate(pg_database中的列)设置为T即可。

当然，在创建新库的时候，还可以选择其他的库做为源库：

```bash
db_test=# create database db_test_2 template db_test;
CREATE DATABASE
db_test=#
```

但是，要求不能有其他连接连接到模板库，否则会报错：

```bash
db_test=# create database db_test_2 template db_test;
ERROR:  source database "db_test" is being accessed by other users
DETAIL:  There is 1 other session using the database.
db_test=#
```
