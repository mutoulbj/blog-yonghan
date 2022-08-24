---
title: "PostgreSQL-9.5 配置"
date: "2017-06-15"
draft: false
tags: ["database"]
keywords: ["postgresql", "安装"]
categories: "技术"
---

操作系统：`Ubuntu 16.04`

## 安装PostgreSQL   
```bash
sudo apt-get install postgresql libpq-dev postgresql-client postgresql-client-common
```
安装完成后，PostgreSQL默认已经启动，开始使用之前需要创建一个供其使用的用户。首先切换到`postgres`用户下。  

```bash
sudo -i -u postgres
```

创建一个与系统登录用户相同的用户:  
```bash
createuser username -P  --interactive
```

切换回登录的用户，然后创建一个新的数据库。  

```bash
createdb test_db
```

到此，就可以使用该用户名和密码来使用PostgreSQL数据库了。测试：  

```bash
psql test_db
```

