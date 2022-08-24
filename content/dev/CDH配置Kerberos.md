---
title: "CDH6.3.1 配置 Kerberos"
date: "2021-12-17"
draft: false
tags: ["bigdata"]
keywords: ["CDH", "Kerberos"]
categories: "技术"
---

## 安装KDC

```bash
yum install krb5-server krb5-libs krb5-auth-dialog krb5-workstation
```

## 修改配置

```bash
# vi /etc/krb5.conf
# Configuration snippets may be placed in this directory as well
includedir /etc/krb5.conf.d/

[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults]
 dns_lookup_realm = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true
 rdns = false
 pkinit_anchors = FILE:/etc/pki/tls/certs/ca-bundle.crt
 default_realm = CDH6.COM
 #default_ccache_name = KEYRING:persistent:%{uid}
 dns_lookup_realm = false
 dns_lookup_kdc = false

[realms]
 CDH6.COM = {
  kdc = slave1
  admin_server = slave1
 }

[domain_realm]
.cdh5.com = CDH6.COM
cdh5.com = CDH6.COM

# 修改 /var/kerberos/krb5kdc/kadm5.acl 配置
*/admin@CDH6.COM  *

# 修改/var/kerberos/krb5kdc/kdc.conf
[kdcdefaults]
 kdc_ports = 88
 kdc_tcp_ports = 88

[realms]
 CDH6.COM = {
  #master_key_type = aes256-cts
  acl_file = /var/kerberos/krb5kdc/kadm5.acl
  dict_file = /usr/share/dict/words
  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
  #supported_enctypes = aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal camellia256-cts:normal camellia128-cts:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
  supported_enctypes = des3-hmac-sha1:normal arcfour-hmac:normal camellia256-cts:normal camellia128-cts:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
  max_life=1d
  max_renewable_life=7d
 }
```

## 创建 Kerberos 数据库

```bash
kdb5_util create -r CDH6.COM -s
```

## 创建 Kerberos 管理账号

```bash
kadmin.local
addprinc admin/admin@CDH6.COM
```

![kerberos-admin](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-kerberos-admin.jpg)

## Kerberos 添加到自启动

```bash
systemctl enable krb5kdc
systemctl enable kadmin
systemctl start krb5kdc
systemctl start kadmin
```

## 为集群所有节点安装 Kerberos 客户端

```bash
yum -y install krb5-libs krb5-workstation
# 集群 master 节点安装 openldap-clients
yum -y install openldap-clients
# KDC Server上的krb5.conf文件拷贝到所有Kerberos客户端
rsync -av /etc/krb5.conf root@cdh6-m:/etc/
```

## CDH6 启用 Kerberos

```bash
# 添加集群管理员账号
kadmin.local
addprinc cloudera-scm/admin@CDH6.COM
```

![cdh-kerberos-admin](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-cdh-kerberos-admin.jpg)

## 进入集群 Web 后台操作，步骤如图

![kerberos-cdh-1](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-kerberos-cdh-1.jpg)

![kerberos-cdh-2](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-kerberos-cdh-2.jpg)

![kerberos-cdh-3](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-kerberos-cdh-3.jpg)

![kerberos-cdh-5](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-20211108-kerberos-cdh-5.jpg)

![kerberos-cdh-6](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-kerberos-cdh-6.jpg)

![kerberos-cdh-7](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-kerberos-cdh-7.jpg)

![kerberos-cdh-8](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-kerberos-cdh-8.jpg)

![kerberos-cdh-9](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-kerberos-cdh-9.jpg)

![kerberos-cdh-10](https://mutoulbj.oss-cn-zhangjiakou.aliyuncs.com/uPic/20211108-kerberos-cdh-10.jpg)

