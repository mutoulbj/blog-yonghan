---
title: "CentOS上部署Django"
date: "2017-07-10"
draft: false
tags: ["program"]
keywords: ["centos", "python", "django", "部署"]
categories: ["技术"]
---

### 搭建Python环境

一般云服务（如阿里云）的CentOS系统默认的Python版本较低，还停留在2.6。如果是这样，首先需要搭建较高版本的Python环境。具体可以参见翻译的博文[在CentOS 6.4上设置Python2.7.6和3.3.3环境](https://mutoulbj.com/blog/4)。

依次执行以下命令：

```bash
yum install -y update  # 更新内置程序
yum groupinstall -y development  # 安装所需的development tools
yum install -y zlib-devel openssl-devel sqlite-devel bzip2-devel  # 安装附加包
yum install xz-libs  # 安装XZ解压库(可选)

wget http://www.python.org/ftp/python/2.7.6/Python-2.7.6.tar.xz  # 下载源码包

# 解压源码包,分为两步
xz -d Python-2.7.6.tar.xz
tar -xvf Python-2.7.6.tar

# 编译与安装,先进入源码目录
cd Python-2.7.6
./configure --prefix=/usr/local
make
make altinstall

# 配置virtualenv虚拟环境
wget --no-check-certificate https://pypi.python.org/packages/source/s/setuptools/setuptools-1.4.2.tar.gz
tar -xvf setuptools-1.4.2.tar.gz
cd setuptools-1.4.2
python2.7 setup.py install
curl https://raw.githubusercontent.com/pypa/pip/master/contrib/get-pip.py | python2.7 -
pip install virtualenv

# 创建项目所需的虚拟环境venv
virtualenv venv --python=`which python2.7`

# 修改.bashrc,在该系统用户登录之后自动激活虚拟环境。
# 在.bashrc下增加以下命令
source ~/venv/bin/activate

```



### 安装数据库MySQL

使用yum源直接安装的版本较低，一般需要安装较高版本（5.5及以上）。

```bash
# 添加yum源
## Remi Dependency on CentOS 5 and Red Hat (RHEL) 5 ##
rpm -Uvh http://dl.fedoraproject.org/pub/epel/5/i386/epel-release-5-4.noarch.rpm

## CentOS 5 and Red Hat (RHEL) 5 ##
rpm -Uvh http://rpms.famillecollet.com/enterprise/remi-release-5.rpm

rpm -Uvh http://dl.fedoraproject.org/pub/epel/6/i386/epel-release-6-8.noarch.rpm
rpm -Uvh http://rpms.famillecollet.com/enterprise/remi-release-6.rpm

# 检查可用的MySQL版本
yum --enablerepo=remi,remi-test list mysql mysql-devel mysql-server

# 安装MySQL
yum --enablerepo=remi,remi-test install mysql mysql-server

# 修改/etc/my.conf,修改或者添加以下配置,支持unicode全字符(即支持emoji)
[client]
default-character-set = utf8mb4
[mysqld]
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
default-storage-engine = INNODB

# 启动MySQL
service mysqld start

# 检查配置是否正确
mysql -u root -p   # 回车，不需要输入密码，此时还未设置root密码
mysql> SHOW VARIABLES WHERE Variable_name LIKE 'character\_set\_%' OR Variable_name LIKE 'collation%';

# 如果看到以下结果说明配置正确
+--------------------------+--------------------+
| Variable_name            | Value              |
+--------------------------+--------------------+
| character_set_client     | utf8mb4            |
| character_set_connection | utf8mb4            |
| character_set_database   | utf8mb4            |
| character_set_filesystem | binary             |
| character_set_results    | utf8mb4            |
| character_set_server     | utf8mb4            |
| character_set_system     | utf8               |
| collation_connection     | utf8mb4_general_ci |
| collation_database       | utf8mb4_unicode_ci |
| collation_server         | utf8mb4_unicode_ci |
+--------------------------+--------------------+

# 查看用户信息
mysql> SELECT user,host,password FROM mysql.user;
+------+--------------+----------+
| user | host         | password |
+------+--------------+----------+
| root | localhost    |          |
| root | iz2853tmsqfz |          |
| root | 127.0.0.1    |          |
|     | localhost    |          |
|     | iz2853tmsqfz |          |
+------+--------------+----------+

# 设置root用户密码
mysqladmin -u root password 'password'

# 使用root用户登录后创建新用户
mysql> CREATE USER 'demouser'@'localhost' IDENTIFIED BY 'demopassword';

# 授权
mysql> GRANT ALL PRIVILEGES ON demodb.* to demouser@localhost;
mysql> FLUSH PRIVILEGES;

# 使用新创建的用户登录后创建数据库
mysql> CREATE DATABASE demodb;

```



### 拉取项目代码并安装所需包

```bash
# 安装数据库MySQL
yum install mysql

# 拉取项目代码(示例使用git),假设项目名为proj
git clone 代码库地址

# 安装requirements.txt中所有的包
pip install -r requirements.txt
```

*注意* 如果出现以下错误：

```bash
_mysql.c:2654: error: '_mysql_ResultObject' has no member named 'converter'
    _mysql.c:2654: error: initializer element is not constant
    _mysql.c:2654: error: (near initialization for '_mysql_ResultObject_memberlist[0].offset')
    _mysql.c:2661: error: '_mysql_ResultObject' has no member named 'has_next'
    _mysql.c:2661: error: initializer element is not constant
    _mysql.c:2661: error: (near initialization for '_mysql_ResultObject_memberlist[1].offset')
    _mysql.c: In function '_mysql_ConnectionObject_getattro':
    _mysql.c:2680: error: '_mysql_ConnectionObject' has no member named 'open'
    error: command 'gcc' failed with exit status 1
```

一般上述错误可以使用`yum install mysql-devel` 解决，但是由于这里添加了源来安装MySQL，所以版本会不正确，出现以下错误：

```bash
Error: Package: mysql-devel-5.1.73-7.el6.x86_64 (base)
           Requires: mysql = 5.1.73-7.el6
           Installed: mysql-5.5.52-1.el6.remi.x86_64 (@remi)
               mysql = 5.5.52-1.el6.remi
           Available: mysql-5.1.73-7.el6.x86_64 (base)
               mysql = 5.1.73-7.el6
 You could try using --skip-broken to work around the problem
 You could try running: rpm -Va --nofiles --nodigest
```

解决方法：

```bash
yum --enablerepo=remi,remi-test install mysql-devel
```

### 配置gunicorn

```bash
# gunicorn_start.sh
#!/bin/bash

NAME="demo"
DJANGODIR=/path/to/your/project/
SOCKFILE=/tmp/gunicorn.sock
USER=user
GROUP=group
NUM_WORKERS=2
DJANGO_SETTINGS_MODULE=demo.settings
DJNAGO_WSGI_MODULE=demo.wsgi

cd $DJANGODIR
source /path/to/your/venv/bin/activate
export DJANGO_SETTINGS_MODULE=$DJANGO_SETTINGS_MODULE
export PYTHONPATH=$DJANGODIR:$PYTHONPATH

exec /path/to/your/venv/bin/gunicorn demo.wsgi:application \
    --name $NAME \
    --workers $NUM_WORKERS \
    --bind=unix:$SOCKFILE \
    --log-level=debug \
    --log-file=/path/to/logs/bootcamp_gunicorn.log
```

### 配置supervisor

```bash
pip install supervisor

echo_supervisord_conf > /path/to/conf/supervisord.conf

# 将以下配置加入supervisord.conf中
[program:demo]
command = sh /path/to/gunicorn_start.sh
user = user
stdout_logfile = /path/to/logs/gunicorn_supervisor.log
redirect_stderr = true
environment=LANG=en_US.UTF-8,LC_ALL=en_US.UTF-8

# 启动supervisord
supervisord -c /path/to/supervisord.conf

# 重新加载配置
supervisorctl -c /path/to/supervisord.conf reload

# 重启程序
supervisorctl -c /path/to/supervisord.conf restat demo

# 查看程序运行状态
supervisorctl -c /path/to/supervisord.conf status demo
```



### 配置Nginx

```bash
# 安装Nginx
yum install nginx

# /etc/nginx/conf.d/demo.conf文件中写入以下配置
upstream demo_server {
    server unix:/path/to/gunicorn.sock fail_timeout=0;
}

server {

      listen       8888;
      server_name  example.com;
      access_log   /path/to/logs/nginx/access.log;
      error_log    /path/to/logs/nginx/error.log;

      location  /static/ {
          root /path/to/demo;
      }

      location  / {
          proxy_redirect        off;
          proxy_set_header      Host             $host;
          proxy_set_header      X-Real-IP        $remote_addr;
          proxy_set_header      X-Forwarded-For  $proxy_add_x_forwarded_for;
          client_max_body_size  10m;

          if (!-f $request_filename) {
              proxy_pass http://demo_server;
              break;
          }
      }

  }

  # 启动/重启Nginx
  /etc/init.d/nginx start|restart

  # reload 配置
  /etc/init.d/nginx reload
```

### 数据表创建与静态文件处理

```bash
# 创建数据库
mysql> create database demo;

# migrate
python manage.py migrate

# collectstatic
python manage.py collectstatic
```