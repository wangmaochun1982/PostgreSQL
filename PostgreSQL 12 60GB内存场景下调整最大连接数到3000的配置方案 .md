PostgreSQL 12 60GB内存场景下调整最大连接数到3000的配置方案 

# 需修改的postgresql.conf核心参数

 

max_connections：最大连接数核心参数，默认值通常为100，直接调整为3000即可。该参数为静态参数，调整后必须重启实例生效。


 
