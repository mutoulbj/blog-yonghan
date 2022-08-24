---
title: "PostgreSQL间迁移"
date: "2021-12-17"
draft: false
tags: ["database"]
keywords: ["postgresql", "迁移"]
categories: "技术"
---

一般做 PostgreSQL 数据库之间的迁移的做法为：

1. pg_dump 到文件
2. 传到需要迁移到的 pg 节点
3. 导入数据

当两台机器可以相互访问且数据量不大时，直接连接并导入最方便。

```bash
pg_dump -C -h localhost -U pg_user db_name  | psql postgresql://remote_pg_user:pg_password@remote_host:port/postgres
```

当需要导入多个数据库或者所有数据库时，简单写个脚本即可。首先需要查出需要迁移的数据库。 以导入所有数据库为例：

```sql
# 不迁移template和postgres
SELECT datname FROM pg_database WHERE NOT datistemplate AND datname <> 'postgres'";
```

完整脚本：

```bash
dbs=`psql --tuples-only -P format=unaligned -c "SELECT datname FROM pg_database WHERE NOT datistemplate AND datname <> 'postgres'";`

for item in $dbs
do
  pg_dump -C -h localhost -U pg_user $item  | psql postgresql://remote_pg_user:pg_password@remote_host:port/postgres
done
```

