---
title: "解决 Mac 下 git clone 很慢的问题"
date: "2021-12-16"
draft: false
tags: ["科学上网"]
keywords: ["git", "mac", "科学上网"]
categories: ["技术"]
---


虽然一直科学上网，但是 git clone 还是一直很慢，原因是终端没有走代理。设置系统代理或者设置 git 代理配置。

## 方法一
```bash
export https_proxy="http://127.0.0.1:7890"
export http_proxy="http://127.0.0.1:7890"
export all_proxy="socks5://127.0.0.1:7890"
```

## 方法二
```bash
git config --global http.proxy http://127.0.0.1:7890
git config --global https.proxy https://127.0.0.1:7890
```