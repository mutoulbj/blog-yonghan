---
title: "MySQL离线安装"
date: "2021-12-17"
draft: false
tags: ["database"]
keywords: ["离线安装", "MySQL"]
category: "技术"
---

## 准备

- 下载MySQL离线安装包，如  `mysql-5.7.11-linux-glibc2.5-x86_64.tar.gz`

## 创建MySQL所需目录并解压安装包到对应目录

```bash
tar -xzvf  mysql-5.7.11-linux-glibc2.5-x86_64.tar.gz -C /usr/local/
mv /usr/local/mysql-5.7.11-linux-glibc2.5-x86_64 /usr/local/mysql
mkdir -p /usr/local/mysql/data /usr/local/mysql/tmp /usr/local/mysql/arch
```

## MySQL配置

```bash
# 配置文件,写入 /usr/local/mysql/my.cnf

[client]
port            = 3306
socket          = /usr/local/mysql/data/mysql.sock
default-character-set=utf8mb4

[mysqld]
port            = 3306
socket          = /usr/local/mysql/data/mysql.sock

skip-slave-start

skip-external-locking
key_buffer_size = 256M
sort_buffer_size = 2M
read_buffer_size = 2M
read_rnd_buffer_size = 4M
query_cache_size= 32M
max_allowed_packet = 16M
myisam_sort_buffer_size=128M
tmp_table_size=32M

table_open_cache = 512
thread_cache_size = 8
wait_timeout = 86400
interactive_timeout = 86400
max_connections = 600

# Try number of CPU's*2 for thread_concurrency
#thread_concurrency = 32

#isolation level and default engine
default-storage-engine = INNODB
transaction-isolation = READ-COMMITTED

server-id  = 1739
basedir     = /usr/local/mysql
datadir     = /usr/local/mysql/data
pid-file     = /usr/local/mysql/data/hostname.pid

#open performance schema
log-warnings
sysdate-is-now

binlog_format = ROW
log_bin_trust_function_creators=1
log-error  = /usr/local/mysql/data/hostname.err
log-bin = /usr/local/mysql/arch/mysql-bin
expire_logs_days = 7

innodb_write_io_threads=16

relay-log  = /usr/local/mysql/relay_log/relay-log
relay-log-index = /usr/local/mysql/relay_log/relay-log.index
relay_log_info_file= /usr/local/mysql/relay_log/relay-log.info

log_slave_updates=1
gtid_mode=OFF
enforce_gtid_consistency=OFF

# slave
slave-parallel-type=LOGICAL_CLOCK
slave-parallel-workers=4
master_info_repository=TABLE
relay_log_info_repository=TABLE
relay_log_recovery=ON

#other logs
#general_log =1
#general_log_file  = /usr/local/mysql/data/general_log.err
#slow_query_log=1
#slow_query_log_file=/usr/local/mysql/data/slow_log.err

#for replication slave
sync_binlog = 500


#for innodb options
innodb_data_home_dir = /usr/local/mysql/data/
innodb_data_file_path = ibdata1:1G;ibdata2:1G:autoextend

innodb_log_group_home_dir = /usr/local/mysql/arch
innodb_log_files_in_group = 4
innodb_log_file_size = 1G
innodb_log_buffer_size = 200M

#根据生产需要，调整pool size
innodb_buffer_pool_size = 2G
#innodb_additional_mem_pool_size = 50M #deprecated in 5.6
tmpdir = /usr/local/mysql/tmp

innodb_lock_wait_timeout = 1000
#innodb_thread_concurrency = 0
innodb_flush_log_at_trx_commit = 2

innodb_locks_unsafe_for_binlog=1

#innodb io features: add for mysql5.5.8
performance_schema
innodb_read_io_threads=4
innodb-write-io-threads=4
innodb-io-capacity=200
#purge threads change default(0) to 1 for purge
innodb_purge_threads=1
innodb_use_native_aio=on

#case-sensitive file names and separate tablespace
innodb_file_per_table = 1
lower_case_table_names=1

[mysqldump]
quick
max_allowed_packet = 128M

[mysql]
no-auto-rehash
default-character-set=utf8mb4

[mysqlhotcopy]
interactive-timeout

[myisamchk]
key_buffer_size = 256M
sort_buffer_size = 256M
read_buffer = 2M
write_buffer = 2M
```

## 创建用户相关

```bash
# 创建dba用户组
groupadd -g 101 dba
# 创建mysqladmin 用户
useradd -u 514 -g dba -G root -d /usr/local/mysql mysqladmin
# 复制环境变量文件(重要)
cp /etc/skel/.* /usr/local/mysql

# 配置环境变量
vi /usr/local/mysql/.bash_profile

# .bash_profile
# Get the aliases and functions

if [ -f ~/.bashrc ]; then
        . ~/.bashrc
fi

# User specific environment and startup programs
export MYSQL_BASE=/usr/local/mysql
PATH=$PATH:$HOME/.local/bin:$HOME/bin:$MYSQL_BASE/bin
export PATH

unset USERNAME

#stty erase ^H
set umask to 022
umask 022
PS1=`uname -n`":"'$USER'":"'$PWD'":>"; export PS1

## end
```

## 配置mysqladmin用户, mysql 服务

```bash
chown  mysqladmin:dba /usr/local/mysql/my.cnf
chmod  640 /usr/local/mysql/my.cnf
chown -R mysqladmin:dba /usr/local/mysql
chmod -R 755 /usr/local/mysql

# 拷贝服务文件到 init.d 目录
cp /usr/local/mysql/support-files/mysql.server /etc/rc.d/init.d/mysql
chmod +x /etc/rc.d/init.d/mysql
# 添加服务
chkconfig --add mysql
chkconfig --level 345 mysql on
```

## 安装libaio库，初始化db

```bash
yum install -y libaio
# 初始化
su - mysqladmin
bin/mysqld --defaults-file=/usr/local/mysql/my.cnf --user=mysqladmin --basedir=/usr/local/mysql/ --datadir=/usr/local/mysql/data/ --initialize
# 查看初始密码
cat /usr/local/mysql/data/hostname.err |grep password
# 启动mysql
service mysql start
# 修改用户密码
mysql -uroot -p

alter user root@localhost identified by 'passwd';
grant all privileges on *.* to 'root'@'%' identified by 'passwd';
flush privileges;

# 重启
service mysql restart
```

![mysql-set](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-mysql-set.jpg)

## 安装完成

```bash
# 测试
mysql -u root -p
```



