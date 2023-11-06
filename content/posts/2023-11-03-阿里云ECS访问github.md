---
title: "阿里云 ECS 访问 GitHub"
date: "2023-11-03"
draft: false
tags: ["program"]
keywords: ["阿里云", "ecs", "GitHub"]
categories: ["技术"]
---

使用阿里云 ECS 访问 Github 和拉取代码时，速度非常慢，等于不可用。 

**注** 本文解决方案适用于墙内所有云服务器。

## 解决 GitHub 访问问题

阻碍 GitHub 访问的一般手段是 DNS 污染，可以通过修改`hosts`的方式暂时缓解。

1. 访问 [ipaddress.com](https://www.ipaddress.com/),获取`github.com`的 IP 地址；
2. 将记录添加进服务器`/etc/hosts`，如：`echo "140.82.112.4 github.com" >> /etc/hosts` 。

![github-ipaddress](https://img.mutoulbj.com/blog/202311github-ipaddress.png)

## 使用 SSH 代理

本方法需要一台访问可快速访问 GitHub 的云主机（境外云主机）用作代理，将所有 git 操作通过该主机中转。

1. 配置好阿里云主机与境外云主机的免密登录；

2. 在`~/.ssh/config` 中添加以下配置内容：

   ```shell
   Host github.com
       ProxyCommand ssh 用户名@境外云主机 nc %h %p
   ```

3. 确保境外云主机已安装`nc`,并可正常执行。

