```bash

# 关闭防火墙
systemctl disable --now firewalld
# 关闭selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/configsetenforce 0
# 确认时间同步
systemctl status chronyd
chronyc sources
# 安装程序
dnf install -y tar gzip curl net-tools telnet

dnf install -y readline-devel gcc make zlib-devel perl-devel perl-ExtUtils-Embed python3-devel

dnf install  libicu-devel lz4 lz4-devel libzstd  bison flex
yum install libzstd libzstd-devel -y
yum install -y krb5-devel
yum install openssl openssl-devel
yum install pam-devel
yum install libxml2 libxml2-devel
yum install libxslt-devel
yum install -y openldap openldap-devel
yum install libuuid-devel
dnf install systemd-devel
yum install gettext
yum install tcl tcl-devel
yum install perl-IPC-Run
perl -MCPAN -e 'install IPC::Run'
pip install gssapi

mkdir /opt
mkdir /data
cd /data
mkdir pgdata

cd /opt
wget https://ftp.postgresql.org/pub/source/v18.3/postgresql-18.3.tar.gz
tar -xf postgresql-18.3.tar.gz
cd postgresql-18.3/

cd /usr/local
mkdir pg18
./configure   --enable-rpath  --prefix=/usr/local/pg18 --includedir=/usr/local/pg18/include  --datadir=/data/pgdata/ --with-lz4 --with-zstd --enable-tap-tests --with-icu  --with-perl --with-python --with-tcl --with-openssl --with-pam --with-gssapi --enable-nls --with-uuid=e2fs \
--with-libxml --with-libxslt --with-ldap --with-systemd 

# 构建所有内容，包括官方扩展包，不包含文档
make world-bin -j4
# 安装所有内容，包括官方扩展包，不包含文档
make install-world-bin

root@devops10:/usr/local/pg18# groupadd -g 701 postgres
root@devops10:/usr/local/pg18# useradd -g 701 -u 701 -m postgres

groupadd postgres
useradd -g postgres postgres
echo "post#2026~"|passwd --stdin postgres

 cat >>/etc/sysctl.conf << "EOF"
#postgresql set
fs.file-max = 76724200
kernel.sem = 10000 10240000 10000 1024
kernel.shmmni = 4096
kernel.shmall = 1073741824
kernel.shmmax = 4398046511104
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.wmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_max = 1048576
fs.aio-max-nr = 40960000
vm.dirty_ratio = 20
vm.dirty_background_ratio = 3
vm.dirty_writeback_centisecs = 100
vm.dirty_expire_centisecs = 500
vm.min_free_kbytes = 524288
vm.swappiness = 0
vm.overcommit_memory = 2
vm.overcommit_ratio = 75
kernel.io_uring_disabled = 0
EOF
/sbin/sysctl -p

cat >> /etc/security/limits.conf << "EOF"
#postgresql set
postgres soft nofile 1048576
postgres hard nofile 1048576
postgres soft nproc 131072
postgres hard nproc 131072
postgres soft stack 10240
postgres hard stack 32768
postgres soft core 6291456
postgres hard core 6291456
EOF

su - postgres
cat >> /home/postgres/.bash_profile << "EOF" #PostgreSQL settings export PGPORT=5432 export PGUSER=postgres export PGHOME=/usr/local/pg18 export PGDATA=/data/pgdata/pgdata18 export LD_LIBRARY_PATH=$PGHOME/lib export MANPATH=$PGHOME/share/man export PATH=$PGHOME/bin:$PATH export LANG="en_US.UTF-8" EOF

source ~/.bash_profile

postgres --version
pg_config

su - root
chown -R postgres:postgres /usr/local/pg18
chown -R postgres:postgres /data/pgdata

su - postgres
initdb -D /data/pgdata -k -E UTF8 -U postgres -W

cat >/data/pgdata/pgdata18/postgresql.conf << "EOF"

#add line
listen_addresses = '*'
port = 5432
shared_buffers = 4GB
effective_cache_size = 12GB
maintenance_work_mem = 1GB
work_mem = 30MB
huge_pages = off
maintenance_work_mem = 256MB
temp_buffers = 256MB
max_connections = 500
checkpoint_completion_target = 0.9
wal_buffers = 16MB
min_wal_size = 4GB
max_wal_size = 64GB
wal_log_hints = on
wal_keep_size = 1000
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
max_worker_processes = 8
max_parallel_workers_per_gather = 4
max_parallel_workers = 8
max_parallel_maintenance_workers = 4
wal_level = replica
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_file_mode = 0600
log_rotation_age = 1d
log_rotation_size = 0
log_truncate_on_rotation = off
archive_mode = on
archive_command = 'test ! -f /data/pgdata/pgdata18/archive/%f && cp %p /data/pgdata/pgdata18/archive/%f'

EOF

pg_ctl -D /data/pgdata/pgdata18 -l logfile start
pg_ctl stop
pg_ctl restart

su - root

cat > /usr/lib/systemd/system/postgres.service << "EOF"

[Unit]
Description=PostgreSQL database server
After=network.target
[Service]
Type=forking
User=postgres
Group=postgres
Environment=PGPORT=5432
Environment=PGDATA=/data/pgdata/pgdata18/
OOMScoreAdjust=-1000
ExecStart=/usr/local/pg18/bin/pg_ctl start -D $PGDATA
ExecStop=/usr/local/pg18/bin/pg_ctl stop -D $PGDATA -s -m fast
ExecReload=/usr/local/pg18/bin/pg_ctl reload -D $PGDATA -s
TimeoutSec=300
[Install]
WantedBy=multi-user.target

EOF

chmod +x /usr/lib/systemd/system/postgres.service
systemctl daemon-reload
systemctl enable --now postgres.service
systemctl status postgres.service

systemctl stop postgres.service
```
