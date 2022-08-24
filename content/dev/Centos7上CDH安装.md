---
title: "CentOS7 上安装 CDH6"
date: "2021-12-17"
draft: false
tags: ["bigdata"]
keywords: ["centos", "CDH6"]
categories: "技术"
---

## 准备工作

自2021年1月31日开始，CDH加进了Cloudera的付费墙，不再提供免费的安装包下载。CDH6.3.1 安装需要准备以下文件，找到下载并上传至服务器：

- CDH-6.3.1-1.cdh6.3.1.p0.1470567-el7.parcel
- CDH-6.3.1-1.cdh6.3.1.p0.1470567-el7.parcel.sha1
- cm6.3.1-redhat7.tar.gz
- jdk-8u291-linux-x64.tar.gz
- manifest.json
- mysql-5.7.11-linux-glibc2.5-x86_64.tar.gz
- mysql-connector-java-5.1.49.jar

## 操作系统配置

### 主机名与hosts

每台机器都执行以下操作

```bash
echo "[hostname]" > /etc/hostname

vi /etc/hosts
# 添加以下内容
192.168.239.1 cdh6-m      # master节点
192.168.239.2 cdh6-s1     # slave 1
192.168.239.3 cdh6-s2     # slave 2
```

重启或退出重新登录使  `hostname` 生效。

### 免密登陆

```bash
# 生成密钥
ssh-keygen
# 使用ssh-copy-id将密钥同步到各个机器,每台机器分别执行,过程需要输入对应机器登录密码
ssh-copy-id root@cdh6-m
ssh-copy-id root@cdh6-s1
ssh-copy-id root@cdh6-s2
```

### 关闭防火墙

```bash
systemctl stop firewalld.service
systemctl disable firewalld.service
```

### 禁用SELinux

```bash
# 查看状态
sestatus
# 修改/etc/selinux/config 文件,将 SELINUX设为disabled
vi /etc/selinux/config

SELINUX=disabled
```

![selinus-status](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211105-20211105-selinus-status.jpg)

![set-selinux](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211105-20211105-selinus-status.jpg)

重启，使其生效。

注：配置文件相关的修改，可在 `master` （任意一台机器）修改完，使用 `rsync` 同步到其他机器，这里可如下操作:

```bash
rsync -av /etc/selinux/config root@cdh6-s1:/etc/selinux/config
```

### 关闭 swap

```bash
# 删除 swap 中内容
swapoff -a
# 移除 swap 挂载,注释掉 /etc/fstab 中 swap 一行
vi /etc/fstab

#
# /etc/fstab
# Created by anaconda on Wed May 29 08:59:47 2019
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=092a9846-1fd3-4a82-b577-8208cf71199b /boot                   xfs     defaults        0 0
# /dev/mapper/centos-swap swap                    swap    defaults        0 0
```

重启使其生效，重启后使用 `free -h` 查看是否已成功。

### 关闭系统透明大页 THP

```bash
grubby --update-kernel=ALL --args="transparent_hugepage=never"
```

配置过后需要重启生效，使用以下命令检查，若结果为 `always[never]` 说明配置成功。

```bash
cat /sys/kernel/mm/*transparent_hugepage/enabled

always [never]
```

###  修改系统资源限定（打开文件数）

```bash
#  查看当前限定
ulimit -a
# 修改 /etc/security/limits.conf 中限定数,如下,*可使用特定的用户代替表示针对特定的用户设置
* soft nofile 65536
* hard nofile 65536
```

![ulimit](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211105-ulimit.jpg)

![limits](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211105-limits.jpg)

 重启后使设置生效。

### 设置时区

```bash
timedatectl set-timezone  Asia/Shanghai
```

### 配置时钟同步

以 `master` 为标准，其他机器从它同步时钟。

- 配置 `master` 从阿里云时钟服务器同步时间（若无外网，将 master 节点时间设置正确即可）

  ```bash
  # 修改 /etc/ntp.conf 配置文件,若没有则新建
  vi /etc/ntp.conf

  server ntp.aliyun.com
  ```

- 其他节点配置从 `master` 同步

  ```bash
  vi /etc/ntp.conf
  
  server cdh5-m
  ```

###  安装Java

```bash
mkdir /usr/java
tar -xzvf jdk-8u291-linux-x64.tar.gz -C /usr/java

# /etc/profile 中添加以下行
export JAVA_HOME=/usr/java/jdk1.8.0_291
export JRE_HOME=$JAVA_HOME/jre
export PATH=$JAVA_HOME/bin:$PATH

# 将 mysql-connector 包放到 /usr/share/java 目录下
cp mysql-connector-java-5.1.49-bin.jar /usr/share/java/mysql-connector-java.jar
```

## 安装MySQL

参考 [离线安装MySQL](https://mutoulbj.com/#/blog/15)

## CDH 安装

### 配置本地Yum源

-  httpd 服务

  ```bash
  # 安装
  yum install -y httpd
  # 开启
  systemctl httpd start
  ```

- 将 cdh 所有安装包放到 `/var/www/html`目录下

  ```bash
  # 创建目录
  mkdir /var/www/html/cm /var/www/html/cdh
  # 解压 cm6.3.1-redhat7.tar.gz 到 /var/www/html/cm 目录下
  tar -xzvf cm6.3.1-redhat7.tar.gz -C /var/www/html/cm
  mv CDH-6.3.1-1.cdh6.3.1.p0.1470567-el7.parcel /var/www/html/cdh
  mv manifest.json /var/www/html/cdh
  ```

  ![cdh-yum-tree-1](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-cdh-yum-tree-1.jpg)

![cdh-httpd](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-cdh-httpd.jpg)

![cdh-yum-cm](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-cdh-yum-cm.jpg)

### 安装CM服务

```bash
# 进入目录 /var/www/html/cm/RPMS/x86_64
rpm -ivh cloudera-manager-daemons-6.3.1-1466458.el7.x86_64.rpm
rpm -ivh cloudera-manager-server-6.3.1-1466458.el7.x86_64.rpm

# 开机自启动
systemctl enable cloudera-scm-server.service

# 数据库初始化
# 进入 mysql 创建 scm 数据库
create database scm;

# 自动初始化 scm 所需数据
/opt/cloudera/cm/schema/scm_prepare_database.sh mysql scm  root [mysql密码]

# 启动 cloudera-scm-server,启动较慢,耐心等待一段时间,或者监控服务日志
systemctl start cloudera-scm-server.service
# 服务日志位置 : /var/log/cloudera-scm-server/cloudera-scm-server.log
```

进入 `http://cdh6-m:7180`，进入界面安装。

### 按界面提示，根据需求进行安装

![cdh-ui-1](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-cdh-ui-1.jpg)

![cdh-ui-2](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-cdh-ui-2.jpg)

![cdh-ui-3](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-cdh-ui-3.jpg)

![cdh-ui-4](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-cdh-ui-4.jpg)

![cdh-ui-5](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-cdh-ui-5.jpg)

![cdh-ui-6](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-cdh-ui-6.jpg)

![cdh-ui-7](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-cdh-ui-7.jpg)

![cdh-ui-8](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-cdh-ui-8.jpg)

![cdh-ui-9](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-cdh-ui-9.jpg)

![cdh-ui-10](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-cdh-ui-10.jpg)

![cdh-ui-11](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-cdh-ui-11.jpg)

![cdh-ui-12](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-cdh-ui-12.jpg)

## 错误解决

### namenode format 失败

```bash
# 错误日志
Failed to format NameNode.

# 解决方法
# namenode节点
rm -rf /dfs/nn
# datanode
rm -rf /dfs/dn
```

### HiveMetaException: Failed to get schema version, Cause:Table 'scm.version' doesn't exist

原因： hive 元数据表设置错误，hive 的元数据表为 hive，需要在 MySQL 中创建对应的 Database。参考：

https://docs.cloudera.com/documentation/enterprise/6/6.3/topics/cdh_ig_hive_schema_tool.html

```bash
# /etc/hive/conf/hive-site.xml 中添加以下配置
<property>
   <name>javax.jdo.option.ConnectionURL</name>
   <value>jdbc:mysql://my_cluster.com:3306/hive1?useUnicode=true&amp;characterEncoding=UTF-8</value>
</property>
<property>
   <name>javax.jdo.option.ConnectionDriverName</name>
   <value>com.mysql.jdbc.Driver</value>
</property>

# 生成 hive 表数据
schematool -dbType mysql -initSchema -passWord <db_user_pswd> -userName <db_user_name>
```

如果依然报错，说明 scm 元数据中配置的值还为旧值，如 scm

```sql
# 手动修改 mysql 中数值
select * from configs where ATTR='hive_metastore_database_name';
# 修改为正确值
update configs set VALUE='hive' where CONFIG_ID=95;
```

![fix-hive-config](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-fix-hive-config.jpg)

### Bad   :    11 under replicated blocks in the cluster. 16 total blocks in the cluster. Percentage under replicated blocks: 68.75%. Critical threshold: 40.00%.

一般原因是因为DataNode节点数不足，如CDH要求DataNode至少为3节点，修复方法有：

- 添加 DataNode 节点

- 强制修改DataNode节点数，比如设置为2节点

  ```bash
  sudo -u hdfs hdfs dfs -setrep -w 2 -R /
  ```

### 节点间网络问题

```bash
# 报错信息
virbr0-nic.1 host network interface(s) appear to be operating at full speed.For 2 host network interface(s), the Cloudera Manager Agent could not determine the duplex mode or interface speed.
```

一般是因为多网卡问题,修改方式参考：

https://stackoverflow.com/questions/36331486/cdh-network-interface-speed-suppress/42965350

```bash
This is because by default CDH Hosts are required to be deployed on servers with 1GB or better networking interfaces. But you can always change the default configurations to meet your server configurations:

1- From Cloudera Manager Navigate to "Hosts -> All Hosts" and then click "Configuration" in this page.

2- In the search bar, search for "Network Interface".

3- Depending on the type of Network you are on, adjust the value of the 2 configuration parameters "Network Interface Expected Link Speed" and "Network Interface Expected Duplex Mode".

4- Deploy the new configuration and restart Cloudera Manager.

# Network Interface Collection Exclusion Regex 改为如下配置
^lo|virbr0-nic|virbr0$
```



