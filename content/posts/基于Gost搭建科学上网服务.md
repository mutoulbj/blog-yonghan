---
title: "基于Gost搭建科学上网服务"
date: "2021-12-17"
draft: false
tags: ["科学上网"]
keywords: ["gost", "科学上网"]
categories: "技术"
ShowBreadCrumbs: true
---

## 准备

VPS一台，目前在用的有以下几个服务商。

- [Vultr](https://www.vultr.com/?ref=7194684)
- [DigitalOcean](https://m.do.co/c/fa7467275ddf)
- [UCloud香港](https://ucloud.cn)

在国外注册商购买的域名一个。

## 服务搭建

### Docker安装

在VPS上安装Docker CE，具体参看文档：

- [CentOS上Docker CE安装](https://docs.docker.com/install/linux/docker-ce/centos/)
- [Ubuntu上Docker CE安装](https://docs.docker.com/install/linux/docker-ce/ubuntu/)

非root用户使用Docker如果遇到错误

```
Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock:
Get http://%2Fvar%2Frun%2Fdocker.sock/v1.40/containers/json: dial unix /var/run/docker.sock: connect: permission denied
```

解决办法如下：

```shell
# 创建docker用户组
sudo groupadd docker
# 将当前用户添加进docker用户组
sudo gpasswd -a ${USER} docker
# 重启docker服务,退出当前用户重新登录即可
service docker restart
```

### 开启TCP BBR

参见文档[开启 TCP BBR 拥塞控制算法](https://github.com/iMeiji/shadowsocks_install/wiki/开启-TCP-BBR-拥塞控制算法)

### 域名和证书配置

设置DNS解析，将域名解析到对应VPS。
使用[certbot](https://certbot.eff.org/instructions) 在服务器上生成 [Let's Encrypt ](https://letsencrypt.org/)证书。使用以下命令以standalone方式生成证书，按照提示输入域名和邮箱等信息。

```shell
$ sudo certbot certonly --standalone
```

## 启动gost服务

使用Docker来部署gost服务，创建如下脚本：

```shell
#!/bin/bash

## 下面的四个参数需要改成你的
DOMAIN="YOU.DOMAIN.NAME"
USER="username"
PASS="password"
PORT=443

BIND_IP=0.0.0.0
CERT_DIR=/etc/letsencrypt
CERT=${CERT_DIR}/live/${DOMAIN}/fullchain.pem
KEY=${CERT_DIR}/live/${DOMAIN}/privkey.pem
sudo docker run -d --name gost \
    -v ${CERT_DIR}:${CERT_DIR}:ro \
    --net=host ginuerzh/gost \
    -L "http2://${USER}:${PASS}@${BIND_IP}:${PORT}?cert=${CERT}&key=${KEY}&probe_resist=code:404&knock=www.google.com"
```

 执行上述脚本，成功后gost服务就部署完成了。可以使用以下命令检查是否正常运行。

 ```shell
curl -v "https://www.google.com" --proxy "https://DOMAIN" --proxy-user 'USER:PASS'
 ```

 ## 自动更新证书

 因为Let's Encrypt的证书半年后会过期，所以要添加定时任务，自动更新证书。使用 `crontab -e` 编辑Crontab定时任务，添加以下内容：

 ```shell
0 0 1 * * /usr/bin/certbot renew --force-renewal
5 0 1 * * /usr/bin/docker restart gost
 ```

## Mac 客户端配置

 首先下载 [程序](https://github.com/ginuerzh/gost/releases)到本地，解压后添加可执行权限，改名为 `gost`。
执行命令:

 ```shell
gost -L ss://aes-128-cfb:passcode@:1984 -F 'https://USER:PASS@DOMAIN:443'
 ```

成功后本地就启动了一个Shadowsocks服务，在ClashX中配置好该本地Shadowsocks服务即可。

## 手机客户端

- iPhone，购买`ShadowRocket` 或者 `Quantumult` ，然后添加HTTPS服务。
- Android，可以使用插件 [ShadowsocksGostPlugin](https://github.com/xausky/ShadowsocksGostPlugin)，我没用过，可以自行测试。

## 参考

- https://github.com/haoel/haoel.github.io
- https://github.com/ginuerzh/gost