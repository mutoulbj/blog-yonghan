---
title: "Greenplum集群安装指南"
date: "2021-12-17"
draft: false
tags: ["bigdata"]
keywords: ["greenplum", "集群", "安装"]
categories: ["技术"]
---

## 硬件及配置

+ CPU >= 8核
+ 内存 >= 16GB
+ 硬盘 建议系统盘和数据盘分开挂载，系统盘 >= 64GB，数据盘根据实际需要配备
+ 网络 建议千兆及以上
+ 集群规模： master 1台，segment 3台，建议上1台standby

## 系统准备
操作系统以及版本： CentOS 64bit 7.8
CentOS下软件依赖：

```bash
apr
apr-util
bash
bzip2
curl
krb5
libcurl
libevent
libxml2
libyaml
zlib
openldap
openssh
openssl
openssl-libs (RHEL7/Centos7)
perl
readline
rsync
R
sed (used by gpinitsystem)
tar
zip
```
备注： 查看系统版本命令
```cat /etc/issue 或者 cat /etc/redhat-release ```

## 主机设置
集群规划，以IP为 192.168.239.215/216/217/218 4台机器为例。
```bash
192.168.239.215 mdw     Master    管理节点
192.168.239.216 sdw1    Segment1  数据节点1
192.168.239.217 sdw2    Segment2  数据节点2
192.168.239.218 sdw3    Segment3  数据节点3
```
修改主机名：编辑各台机器的`/etc/hostname`文件，写入对应的主机名称，修改完成后重启机器。

配置各台机器的`/etc/hosts` 文件，包含各主机的IP和hostname，如：
```bash
192.168.239.215 mdw
192.168.239.216 sdw1
192.168.239.217 sdw2
192.168.239.218 sdw3
```

## 禁用SELinux
*注：* 以下操作均需要root权限
1. 检查SELinux状态
   ```bash
   # sestatus
   SELinux status: disab    led
   ```
   ![Xnip2020-10-20_09-14-13](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211217-2020-10-23-Xnip2020-10-20_09-14-13.jpg)
   如上图，该状态为SELinux开启状态，需要将其改为disabled。

2. 如果SELinux状态不为disabled状态，通过修改`/etc/selinux/config`文件，将其设为禁用
![Xnip2020-10-20_09-20-13](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211217-2020-10-23-Xnip2020-10-20_09-20-13.jpg)
3. 如果系统安装了SSSD，修改`/etc/sssd/sssd.conf`配置文件，将`selinux_provider`设置为`none`，若没有安装则无需操作。
   ```bash
   selinux_provider=none
   ```
4. 重启机器完成设置，并检查是否生效。再次执行`sestatus`，结果如下图则表示配置成功。
![Xnip2020-10-20_09-29-41](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211217-2020-10-23-Xnip2020-10-20_09-29-41.jpg)

## 关闭防火墙软件
1. 查看防火墙状态

    ```bash
    # systemctl status firewalld
    ```
   ![Xnip2020-10-20_10-13-28](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211217-2020-10-23-Xnip2020-10-20_10-13-28.jpg) 上图所示为防火墙运行中。

1. 关闭防火墙，并检查是否关闭成功

    ```bash
    # systemctl stop firewalld.service
    # systemctl disable firewalld.service
    # systemctl status firewalld
    ```
    ![Xnip2020-10-20_10-16-30](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211217-2020-10-23-Xnip2020-10-20_10-16-30.jpg)

## 配置 sysctl.conf
1. 根据实际情况修改 `/etc/sysctl.conf` 文件配置系统参数，具体参数如下：

    ```bash
    # kernel.shmall = _PHYS_PAGES / 2
    kernel.shmall = 2041848
    # kernel.shmmax = kernel.shmall * PAGE_SIZE
    kernel.shmmax = 8363409408
    kernel.shmmni = 4096
    vm.overcommit_memory = 2
    vm.overcommit_ratio = 95

    net.ipv4.ip_local_port_range = 10000 65535
    kernel.sem = 500 2048000 200 4096
    kernel.sysrq = 1
    kernel.core_uses_pid = 1
    kernel.msgmnb = 65536
    kernel.msgmax = 65536
    kernel.msgmni = 2048
    net.ipv4.tcp_syncookies = 1
    net.ipv4.conf.default.accept_source_route = 0
    net.ipv4.tcp_max_syn_backlog = 4096
    net.ipv4.conf.all.arp_filter = 1
    net.core.netdev_max_backlog = 10000
    net.core.rmem_max = 2097152
    net.core.wmem_max = 2097152
    vm.swappiness = 10
    vm.zone_reclaim_mode = 0
    vm.dirty_expire_centisecs = 500
    vm.dirty_writeback_centisecs = 100

    # 注意：主机内存大于64G需要额外添加如下参数：
    vm.dirty_background_ratio = 0
    vm.dirty_ratio = 0
    vm.dirty_background_bytes = 1610612736 # 1.5GB
    vm.dirty_bytes = 4294967296 # 4GB
    # 主机内存小于64G需要额外添加如下参数：
    vm.dirty_background_ratio = 3
    vm.dirty_ratio = 10
    ```

    其中 `kernel.shmall` 和 `kernel.shmmll` 要根据操作系统实际情况配置，具体计算公式如下（上文注释中也有），计算结果四舍五入取**整数**。

    ```bash
    kernel.shmall = _PHYS_PAGES / 2
    kernel.shmmax = kernel.shmall * PAGE_SIZE
    ```

    `_PHYS_PAGES` 和 `PAGE_SIZE` 是操作系统参数，可以通过 `getconf` 命令获取到具体数值。
    ![Xnip2020-10-21_16-22-02](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211217-2020-10-23-Xnip2020-10-21_16-22-02.jpg)
    通过以下命令可以快速获取到两个参数的应该配置的数值。

    ```bash
    $ echo $(expr $(getconf _PHYS_PAGES) / 2)
    $ echo $(expr $(getconf _PHYS_PAGES) / 2 \* $(getconf PAGE_SIZE))
    ```

1. 配置修改完成后，执行 `sysctl -p` 重新加载配置

## 设置资源限定
配置文件 `/etc/security/limits.conf` 中添加以下配置。

```bash
* soft nofile 524288
* hard nofile 524288
* soft nproc 131072
* hard nproc 131072
```

## XFS挂载选项
GP 需要使用XFS的文件系统，RHEL/CentOS 7 和Oracle Linux将XFS作为默认文件系统。如果使用虚机且只有一个分区，则不需此步操作。

例如挂载新xfs步骤：

```bash
[root@mdw ~]# fdisk /dev/sdd
   然后跟着n,p,1提示做下去，最后wq保存退出
[root@mdw ~]# mkfs.xfs /dev/sdd1
# 挂载目录
[root@mdw ~]# mount /dev/sdd1 /data/master
# 设置自动挂载
[root@mdw ~]# vi /etc/fstab
/dev/sdd1 /data xfs rw,noatime,inode64,allocsize=16m 0 0
```

## 磁盘I/O设置
查看磁盘挂载信息 `lsblk`
![Screen Shot 2020-10-22 at 2.39.31 PM](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211217-2020-10-23-Screen%20Shot%202020-10-22%20at%202.39.31%20PM.png)

查看磁盘文件预先读设置，该设置的值需要等于 16384：

```bash
/sbin/blockdev --getra /dev/vda
```
![Xnip2020-10-22_14-42-18](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211217-2020-10-23-Xnip2020-10-22_14-42-18.jpg)

设置磁盘文件预读设置为 16384:

```bash
/sbin/blockdev --setra 16384 /dev/vda
```
![Xnip2020-10-22_14-45-27](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211217-2020-10-23-Xnip2020-10-22_14-45-27.jpg)

以上命令修改的值会在机器重启后失效，所以必须保证每次重启后设置正确，将该命令写进 `/etc/rc.d/rc.local` 文件，并保证该文件有可执行权限，通过命令 `chmod +x /etc/rc.d/rc.local` 添加可执行权限。

![Xnip2020-10-22_14-49-31](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211217-2020-10-23-Xnip2020-10-22_14-49-31.jpg)

磁盘I/O有多种调度算法，GP推荐使用`deadline`，通过以下命令设置，以下示例中的`vda`为磁盘名，根据实际情况进行修改：

```bash
echo deadline > /sys/block/vda/queue/scheduler
```
同样以上方式只能临时生效，CentOS7 永久修改使用以下命令：

```bash
grubby --update-kernel=ALL --args="elevator=deadline"
```

可通过 `grubby --info=ALL`命令查看配置是否生效。
![Xnip2020-10-22_14-58-59](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211217-2020-10-23-Xnip2020-10-22_14-58-59.jpg)

## 关闭系统透明大页(THP)

```bash
grubby --update-kernel=ALL --args="transparent_hugepage=never"
```
该配置修改后需要重启生效，重启后通过以下命令检查是否生效，生效后的结果为 `always [never]` 。

```bash
# cat /sys/kernel/mm/*transparent_hugepage/enabled
always [never]
```

## 禁用 IPC Object Removal
修改 `/etc/systemd/logind.conf` 文件，添加 `RemoveIPC=no` 配置，并重启 `systemd-login` 或者重启机器使其生效。
![Xnip2020-10-22_15-08-29](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211217-2020-10-23-Xnip2020-10-22_15-08-29.jpg)

使用 `service systemd-logind restart` 命令重启 `systemd-login`。

## 修改SSH连接阈值
修改 `/etc/ssh/sshd_config` 修改以下参数：

```bash
MaxStartups 200
MaxSessions 200
```

修改完成后重启ssh：

```bash
# service sshd restart
```

## 设置时间同步
1. 如果有时间服务器，设置Master机器与时间服务器进行同步，如果没有时间服务器，则Master机器将时间修改正确，不需要其他设置，将Segment机器设置与Master同步即可。如果Master需要配置，修改 `/etc/ntp.conf` 文件。

   ```bash
   # IP为时间服务器IP
   server 10.6.220.20
   ```
2. 修改每台Segment机器的 `/etc/ntp.conf` 文件。

   ```bash
   server mdw prefer
   server smdw # 若没有Standby机器可不添加
   ```
3. 若有Standby机器，Standby机器的配置如下。

   ```bash
   server mdw prefer
   server 10.6.220.20 # 若没有时间服务器则不加
   ```

## 创建gpadmin用户
1. 创建 `gpadmin` 组和用户

   ```bash
   # groupadd gpadmin
   # useradd gpadmin -r -m -g gpadmin
   # passwd gpadmin
   ```

1. 切换到 `gpadmin` 用户并生成ssh 密钥

   ```bash
   $ su gpadmin
   $ ssh-keygen -t rsa -b 4096
    Generating public/private rsa key pair.
    Enter file in which to save the key (/home/gpadmin/.ssh/id_rsa):
    Created directory '/home/gpadmin/.ssh'.
    Enter passphrase (empty for no passphrase):
    Enter same passphrase again:
   ```

1. 添加 `gpadmin` 用户  `sudo` 权限
   使用 `visudo` 命令进入权限编辑，去掉 `%wheel` 组的配置的注释：

   ```bash
   %wheel        ALL=(ALL)       NOPASSWD: ALL
   ```

   确保 `%wheel` 的配置包含 `NOPASSWD` 设置，然后将 `gpadmin` 添加进 `wheel` 组。

   ```bash
   # usermod -aG wheel gpadmin
   ```

## Greenplum数据库软件安装
1. 将 `gp-6.10.0-install-package.tar.gz` 压缩包上传到**所有**机器，并解压

   ```bash
   tar -xzvf gp-6.10.0-install-package.tar.gz
   ```
1. 切换到 `gpadmin` 用户，执行以下命令开始安装

   ```bash
   sudo yum install -y ./greenplum-db-6.10.0-2-rhel7-x86_64.rpm
   ```

1. 修改软件文件所属用户和组为 `gpadmin`

   ```bash
   $ sudo chown -R gpadmin:gpadmin /usr/local/greenplum*
   $ sudo chgrp -R gpadmin /usr/local/greenplum*
   ```

## 配置SSH免密登录
1. 登录Master机器，切换到 `gpadmin` 用户
2. 执行 `$ source /usr/local/greenplum-db-<version>/greenplum_path.sh`
3. 使用 `ssh-copy-id` 命令将Master机器的密钥添加到所有其他机器中

   ```bash
   ssh-copy-id sdw1
   ssh-copy-id sdw2
   ssh-copy-id sdw3
   ```
   命令执行过程中会要求输入对应机器的密码。
1. 在Master机器的 `/home/gpadmin` 目录下创建 `hostfile_exkeys` 文件，并将集群中所有机器名写入文件，注意不要有空行和空格

   ```bash
   mdw
   sdw1
   sdw2
   sdw3
   ```

1. 执行以下命令完成免密登录设置

   ```bash
   gpssh-exkeys -f hostfile_exkeys
   ```

## 安装确认
   1. 在Master机器上，切换到 `gpadmin` 用户。
   2. 执行以下命令看是否有错误，是否要求输入密码

      ```bash
      $ gpssh -f hostfile_exkeys -e 'ls -l /usr/local/greenplum-db-6.10.0'
      ```
   3. 如果要求输入密码，则重新执行以下命令

      ```bash
      gpssh-exkeys -f hostfile_exkeys
      ```

 ## 在Master和Standby Master（若有）上创建数据存储区
 依次执行以下命令(root用户执行）

 ```bash
 # mkdir -p /data/master
 # chown gpadmin:gpadmin /data/master
 # source /usr/local/greenplum-db/greenplum_path.sh
 # 以下两行命令如果没有Standby Master则不需要执行
 # gpssh -h smdw -e 'mkdir -p /data/master'
 # gpssh -h smdw -e 'chown gpadmin:gpadmin /data/master'
 ```

## 各个数据节点Segment上创建数据存储区
1. 在Master节点切换到 `gpadmin` 用户；
2. 在 `/home/gpadmin` 目录下创建 `hostfile_gpssh_segonly` 文件，写入各Segment主机名；

   ```bash
   sdw1
   sdw2
   sdw3
   ```
1. 依次执行以下命令;

   ```bash
   # source /usr/local/greenplum-db/greenplum_path.sh
   # gpssh -f /home/gpadmin/hostfile_gpssh_segonly -e 'sudo mkdir -p /data/primary'
   # gpssh -f /home/gpadmin/hostfile_gpssh_segonly -e 'sudo mkdir -p /data/mirror'
   # gpssh -f /home/gpadmin/hostfile_gpssh_segonly -e 'sudo chown -R gpadmin /data/*'
   ```

## 网络性能、磁盘I/O和内存校验
1. 在Master节点使用 `gpadmin` 用户执行；
2. 在 `/home/gpadmin` 目录下创建 `hostfile_gpchecknet_ic1` 和 `hostfile_gpcheckperf` 文件，写入各Segment主机名；
3. 执行以下命令进行校验网络

   ```bash
   $ gpcheckperf -f /home/gpadmin/hostfile_gpchecknet_ic1 -r N -d /tmp > subnet1.out
   ```

1. 执行以下命令校验磁盘I/O和内存

   ```bash
   $ source /usr/local/greenplum-db/greenplum_path.sh
   $ gpcheckperf -f hostfile_gpcheckperf -r ds -D -d /data/primary -d /data/mirror
   
    ====================
    ==  RESULT 2020-10-22T19:22:52.563413
    ====================
   
     disk write avg time (sec): 1048.61
     disk write tot bytes: 197952274432
     disk write tot bandwidth (MB/s): 180.03
     disk write min bandwidth (MB/s): 60.00 [sdw1]
     disk write max bandwidth (MB/s): 60.02 [sdw2]
     -- per host bandwidth --
        disk write bandwidth (MB/s): 60.00 [sdw1]
        disk write bandwidth (MB/s): 60.02 [sdw2]
        disk write bandwidth (MB/s): 60.01 [sdw3]


     disk read avg time (sec): 613.71
     disk read tot bytes: 193067614208
     disk read tot bandwidth (MB/s): 300.02
     disk read min bandwidth (MB/s): 99.99 [sdw2]
     disk read max bandwidth (MB/s): 100.02 [sdw3]
     -- per host bandwidth --
        disk read bandwidth (MB/s): 100.01 [sdw1]
        disk read bandwidth (MB/s): 99.99 [sdw2]
        disk read bandwidth (MB/s): 100.02 [sdw3]


     stream tot bandwidth (MB/s): 39027.80
     stream min bandwidth (MB/s): 12929.20 [sdw3]
     stream max bandwidth (MB/s): 13115.70 [sdw1]
     -- per host bandwidth --
        stream bandwidth (MB/s): 13115.70 [sdw1]
        stream bandwidth (MB/s): 12982.90 [sdw2]
        stream bandwidth (MB/s): 12929.20 [sdw3]
   ```

1. 查看 `subnet1.out`文件和校验输出结果，看各指标是否正常；

## 初始化数据库
1. 在 `/home/gpadmin` 目录下创建 `hostfile_gpinitsystem` 文件，写入各Segment主机名；
2. 配置初始化配置文件

   ```bash
   $ cp $GPHOME/docs/cli_help/gpconfigs/gpinitsystem_config /home/gpadmin/gpinitsystem_config
   ```
   文件内容如下：
   ```bash
   ARRAY_NAME="Greenplum Data Platform"
   SEG_PREFIX=gpseg
   PORT_BASE=6000
   declare -a DATA_DIRECTORY=(/data/primary /data/primary /data/primary)
   MASTER_HOSTNAME=mdw
   MASTER_DIRECTORY=/data/master
   MASTER_PORT=5432
   TRUSTED SHELL=ssh
   CHECK_POINT_SEGMENTS=8
   ENCODING=UNICODE
   MIRROR_PORT_BASE=7000
   declare -a MIRROR_DATA_DIRECTORY=(/data/mirror /data/mirror /data/mirror)
   ```

4. 执行命令进行初始化

```bash
$ gpinitsystem -c gpinitsystem_config -h hostfile_gpinitsystem
```

初始化过程中会询问是否继续，输入 `y` 回车后继续。等待初始化结束看到如下日志信息表示安装成功。
```bash
20201023:10:18:03:018615 gpinitsystem:mdw:gpadmin-[INFO]:-Greenplum Database instance successfully created
20201023:10:18:03:018615 gpinitsystem:mdw:gpadmin-[INFO]:-------------------------------------------------------
20201023:10:18:03:018615 gpinitsystem:mdw:gpadmin-[INFO]:-To complete the environment configuration, please
20201023:10:18:03:018615 gpinitsystem:mdw:gpadmin-[INFO]:-update gpadmin .bashrc file with the following
20201023:10:18:03:018615 gpinitsystem:mdw:gpadmin-[INFO]:-1. Ensure that the greenplum_path.sh file is sourced
20201023:10:18:03:018615 gpinitsystem:mdw:gpadmin-[INFO]:-2. Add "export MASTER_DATA_DIRECTORY=/data/master/gpseg-1"
20201023:10:18:03:018615 gpinitsystem:mdw:gpadmin-[INFO]:-   to access the Greenplum scripts for this instance:
20201023:10:18:03:018615 gpinitsystem:mdw:gpadmin-[INFO]:-   or, use -d /data/master/gpseg-1 option for the Greenplum scripts
20201023:10:18:03:018615 gpinitsystem:mdw:gpadmin-[INFO]:-   Example gpstate -d /data/master/gpseg-1
20201023:10:18:03:018615 gpinitsystem:mdw:gpadmin-[INFO]:-Script log file = /home/gpadmin/gpAdminLogs/gpinitsystem_20201023.log
20201023:10:18:03:018615 gpinitsystem:mdw:gpadmin-[INFO]:-To remove instance, run gpdeletesystem utility
20201023:10:18:03:018615 gpinitsystem:mdw:gpadmin-[INFO]:-To initialize a Standby Master Segment for this Greenplum instance
20201023:10:18:03:018615 gpinitsystem:mdw:gpadmin-[INFO]:-Review options for gpinitstandby
20201023:10:18:03:018615 gpinitsystem:mdw:gpadmin-[INFO]:-------------------------------------------------------
20201023:10:18:03:018615 gpinitsystem:mdw:gpadmin-[INFO]:-The Master /data/master/gpseg-1/pg_hba.conf post gpinitsystem
20201023:10:18:03:018615 gpinitsystem:mdw:gpadmin-[INFO]:-has been configured to allow all hosts within this new
20201023:10:18:03:018615 gpinitsystem:mdw:gpadmin-[INFO]:-array to intercommunicate. Any hosts external to this
20201023:10:18:03:018615 gpinitsystem:mdw:gpadmin-[INFO]:-new array must be explicitly added to this file
20201023:10:18:03:018615 gpinitsystem:mdw:gpadmin-[INFO]:-Refer to the Greenplum Admin support guide which is
20201023:10:18:03:018615 gpinitsystem:mdw:gpadmin-[INFO]:-located in the /usr/local/greenplum-db-6.10.0/docs directory
20201023:10:18:03:018615 gpinitsystem:mdw:gpadmin-[INFO]:-------------------------------------------------------
```
根据日志提示，确保 `source /usr/local/greenplum-db/greenplum_path.sh` 和 `export MASTER_DATA_DIRECTORY=/data/master/gpseg-1` 已经添加进gpadmin用户下的 `.bashrc` 文件。

1. 设置时区

```bash
gpconfig -s TimeZone  #  查看时区，若时区正确(为Asia/Shanghai)则不需要修改
gpconfig -c TimeZone -v 'Asia/Shanghai'  # 将时区修改为Asia/Shanghai
```

## 安装失败后重新安装
可能因为配置错误或者安装出错需要重新安装，可使用 ` gpdeletesystem -d /data/master` 卸载后重新安装。如果 `gpdeletesystem`  无法正常执行，可手工杀死各节点的进程、删除对应存储文件后修改配置重新初始化。

1. Kill掉各节点的greenplum进程，可使用 `ps -ef | grep greenplum` 找到对应进程；
1. Master节点删除 `/data/master/` 目录下的所有文件；
1. Segment节点删除 `/data/primary` 目录下所有文件；
1. 查看各节点 `/tmp` 目录下是否有包含 `.s.PGSQL` 的文件，如 `.s.PGSQL.5432` 和 `.s.PGSQL.5432.lock` ,若存在全部删除；
1. 检查配置是否有错，调整后重新执行 `gpinitsystem -c gpinitsystem_config -h hostfile_gpinitsystem` 进行初始化。

## 集群操作相关命令

1. 查看集群状态 `gpstate -d /data/master/gpseg-1` ；
1. 启动集群 `gpstart` ;
1. 关闭集群 `gpstop` ;
1. 更多管理工具和详情参见[文档](https://gp-docs-cn.github.io/docs/utility_guide/admin_utilities/util_ref.html)。

