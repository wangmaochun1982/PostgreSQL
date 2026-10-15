PostgreSQL 12 60GB内存场景下调整最大连接数到3000的配置方案 

# 需修改的postgresql.conf核心参数

 

* max_connections：最大连接数核心参数，默认值通常为100，直接调整为3000即可。该参数为静态参数，调整后必须重启实例生效。

* shared_buffers：PG共享内存缓存区，用于缓存数据页。常规场景建议为物理内存的25%~30%，但高连接场景下需下调预留足够内存给连接私有内存，

60GB内存下建议设置为8GB ~ 12GB，推荐值10GB。

* work_mem：单个连接执行排序、哈希运算等操作时可申请的私有内存，单连接内多并行操作可申请多份。

3000连接场景下不可设置过高，建议值为1MB ~ 2MB，避免总内存占用超标引发OOM。

* temp_buffers：单个会话缓存临时表的私有内存上限，高连接场景下建议设置为1MB ~ 2MB。

* maintenance_work_mem：VACUUM、建索引等维护操作的专用内存，不与连接数挂钩，60GB内存下建议设置为1GB ~ 2GB即可。

* max_prepared_transactions：若业务使用两阶段提交，该参数值需大于等于max_connections，建议设置为3000；未使用两阶段提交可保留默认值0，同步设为3000可避免后续业务迭代触发报错。


# 调整操作规则 #

* 内存核算规则：所有PG内存总占用不可超过系统物理内存的80%，剩余20%留给操作系统内核缓存及其他进程使用。总占用估算公式：
  
shared_buffers + max_connections * (work_mem + temp_buffers) + maintenance_work_mem < 物理内存 * 0.8

按上述推荐参数计算：10GB + 3000 * (2MB + 2MB) + 2GB = 24GB，远低于60GB*0.8=48GB的安全阈值，冗余充足。

* 参数生效规则：max_connections、shared_buffers、max_prepared_transactions属于静态参数，修改配置文件后必须重启PostgreSQL服务生效；
  
* work_mem、temp_buffers、maintenance_work_mem属于动态参数，执行pg_ctl reload即可生效，无需重启。、



* 系统内核前置调整：调高连接数前需先调整Linux系统限制，避免出现文件句柄不足报错：
  
1.修改/etc/security/limits.conf，为postgres用户设置进程数、文件打开数限制，建议值不低于65535

2. 调整内核kernel.shmmax、kernel.shmall参数，保证可分配的共享内存大小高于shared_buffers设置值


