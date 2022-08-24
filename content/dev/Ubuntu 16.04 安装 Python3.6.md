---
title: "Ubuntu 16.04 安装 Python3.6"
date: "2017-06-11"
draft: false
tags: ["program"]
keywords: ["ubuntu", "python3.6", "安装"]
categories: "技术"
---

## Ubuntu 16.04 安装 Python3.6  

+ 安装所需的依赖  
```bash
apt install build-essential checkinstall libreadline-gplv2-dev \
libncursesw5-dev libssl-dev libsqlite3-dev tk-dev libgdbm-dev libc6-dev \
libbz2-dev openssl
```

+ 下载对应安装包，并解压  
可在官网的[下载页](https://www.python.org/downloads/)找到对应版本的下载链接。
```bash
mkdir $HOME/opt
cd $HOME/opt
curl -O https://www.python.org/ftp/python/3.6.1/Python-3.6.1.tgz
tar -xzf Python-3.6.0rc2.tgz
```

+ 编译安装Python3.6，保持系统默认版本  
```bash
cd Python-3.6.1
./configure --enable-shared --prefix=/usr/local LDFLAGS="-Wl,--rpath=/usr/local/lib"
make altinstall
```

+ 检查是否安装成功  
```bash
python3.6 -V
```

