---
title: "dbt无法连接Spark"
date: "2021-12-17"
draft: false
tags: ["bigdata"]
keywords: ["dbt", "Spark"]
categories: "技术"
---

## dbt 简介

[dbt](https://docs.getdbt.com/docs/introduction) ，Data Build Tool是一个开源命令行工具，用于数据仓库中的数据转换，专注于`ETL`中的`T`。是现代大数据架构中非常热门的组件之一。

## 使用

### 安装

官方提供了4种[安装方式](https://docs.getdbt.com/dbt-cli/install/overview)，分别为

- Homebrew(Mac用户)
- pip
- Docker image
- 源码安装

一般使用`pip`安装较方便快捷，[最佳实践](https://docs.getdbt.com/faqs/install-pip-best-practices)。dbt为每一类的数据（仓）库提供不了不同的`adapter`,选择对应的进行安装， 会自动将依赖`dbt-core`引入。这里是想使用`dbt`连接`Spark`，通过`thrift`协议。

```bash
pip instal dbt-spark[PyHive]
```

安装成功后便可以使用`dbt-cli`命令行，下面是初始化一个`dbt project`的过程。

```bash
[root@mutoulbj spark_demo]# dbt init
08:08:09  Running with dbt=1.1.0
Enter a name for your project (letters, digits, underscore): spark_demo
The profile spark_demo already exists in /root/.dbt/profiles.yml. Continue and overwrite it? [y/N]: Y
Which database would you like to use?
[1] spark

(Don't see the one you want? https://docs.getdbt.com/docs/available-adapters)

Enter a number: 1
host (yourorg.sparkhost.com): mcdp1
[1] odbc
[2] http
[3] thrift
Desired authentication method option (enter a number): 3
port [443]: 10000
schema (default schema that dbt will build objects in): dbt_example
threads (1 or more) [1]: 1
08:08:35  Profile spark_demo written to /root/.dbt/profiles.yml using target's profile_template.yml and your supplied values. Run 'dbt debug' to validate the connection.
08:08:35
Your new dbt project "spark_demo" was created!

For more information on how to configure the profiles.yml file,
please consult the dbt documentation here:

  https://docs.getdbt.com/docs/configure-your-profile

One more thing:

Need help? Don't hesitate to reach out to us via GitHub issues or on Slack:

  https://community.getdbt.com/

Happy modeling!
```

生成的project目录结构如下：

```bash
[root@mutoulbj spark_demo]# tree
.
├── logs
│   └── dbt.log
└── spark_demo
    ├── analyses
    ├── dbt_project.yml
    ├── logs
    │   └── dbt.log
    ├── macros
    ├── models
    │   └── example
    │       ├── my_first_dbt_model.sql
    │       ├── my_second_dbt_model.sql
    │       └── schema.yml
    ├── README.md
    ├── seeds
    ├── snapshots
    └── tests
```

### 问题与排查解决

使用`dbt debug `命令可以验证配置是否正确，数据（仓）库是否可连接。这时候， 问题来了， 一直报如下错误：

```bash
[root@mutoulbj spark_demo]# dbt debug
06:33:04  Running with dbt=1.1.0
dbt version: 1.1.0
python version: 3.9.10
python path: /usr/local/bin/python3.9
os info: Linux-3.10.0-1160.42.2.el7.x86_64-x86_64-with-glibc2.17
Using profiles.yml file at /root/.dbt/profiles.yml
Using dbt_project.yml file at /root/spark_demo/dbt_project.yml

Configuration:
  profiles.yml file [OK found and valid]
  dbt_project.yml file [OK found and valid]

Required dependencies:
 - git [OK found]

Connection:
  host: 192.168.51.194
  port: 10000
  cluster: None
  endpoint: None
  schema: dbt_example
  organization: 0
  Connection test: [ERROR]

1 check failed:
dbt was unable to connect to the specified database.
The database returned the following error:

  >Runtime Error
  Database Error
    failed to connect

Check your database credentials and try again. For more information, visit:
https://docs.getdbt.com/docs/configure-your-profile
```

从错误信息来看，在连接Spark的时候出错了，但没有具体信息。只有进行排查。

- 使用beeline进行连接验证Spark Thift服务是否正常 ✔︎
- 换个环境，在本机（Mac）上验证是否OK ✔︎
- 重装dbt再尝试 ✘

通过以上排查，依然没有头绪，看源码，在`dbt-spark/dbt/adapters/spark/connections.py`找到相关的代码。

```python
elif creds.method == SparkConnectionMethod.THRIFT:
    cls.validate_creds(creds, ["host", "port", "user", "schema"])

    if creds.use_ssl:
        transport = build_ssl_transport(
            host=creds.host,
            port=creds.port,
            username=creds.user,
            auth=creds.auth,
            kerberos_service_name=creds.kerberos_service_name,
        )
        conn = hive.connect(thrift_transport=transport)
    else:
        conn = hive.connect(
            host=creds.host,
            port=creds.port,
            username=creds.user,
            auth=creds.auth,
            kerberos_service_name=creds.kerberos_service_name,
        )  # noqa
    handle = PyhiveConnectionWrapper(conn)
```

可见在调用`hive.connect()`方法时，没有捕获异常并透出。要想知道具体原因，可以在服务器中进行调试。

```bash
[root@mutoulbj ~]# python3
Python 3.9.10 (main, May 27 2022, 10:27:15)
[GCC 4.8.5 20150623 (Red Hat 4.8.5-44)] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> from pyhive import hive
>>> conn = hive.connect(host='192.168.51.194',port=10000)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/usr/local/lib/python3.9/site-packages/pyhive/hive.py", line 104, in connect
    return Connection(*args, **kwargs)
  File "/usr/local/lib/python3.9/site-packages/pyhive/hive.py", line 243, in __init__
    self._transport.open()
  File "/usr/local/lib/python3.9/site-packages/thrift_sasl/__init__.py", line 84, in open
    raise TTransportException(type=TTransportException.NOT_OPEN,
thrift.transport.TTransport.TTransportException: Could not start SASL: b'Error in sasl_client_start (-4) SASL(-4): no mechanism available: No worthy mechs found'
```

Bingo，到这原因就清楚了，`no mechanism available: No worthy mechs found`,sasl相关的依赖包有缺失。

查看当前服务器已有的相关依赖:

```bash
[root@mutoulbj ~]# rpm -qa | grep cyrus
cyrus-sasl-lib-2.1.26-24.el7_9.x86_64
cyrus-sasl-devel-2.1.26-24.el7_9.x86_64
cyrus-sasl-2.1.26-24.el7_9.x86_64
```

确实没有相关的加密算法库，安装并再次检查依赖：

```bash
[root@mutoulbj ~]# yum install cyrus-sasl-devel cyrus-sasl-gssapi cyrus-sasl-md5 cyrus-sasl-plain
[root@mutoulbj ~]# rpm -qa | grep cyrus
cyrus-sasl-gssapi-2.1.26-24.el7_9.x86_64
cyrus-sasl-lib-2.1.26-24.el7_9.x86_64
cyrus-sasl-plain-2.1.26-24.el7_9.x86_64
cyrus-sasl-devel-2.1.26-24.el7_9.x86_64
cyrus-sasl-2.1.26-24.el7_9.x86_64
cyrus-sasl-md5-2.1.26-24.el7_9.x86_64
```

再次运行`dbt debug`，成功。

```bash
[root@mutoulbj spark_demo]# dbt debug
08:08:51  Running with dbt=1.1.0
dbt version: 1.1.0
python version: 3.9.10
python path: /usr/local/bin/python3.9
os info: Linux-3.10.0-1160.42.2.el7.x86_64-x86_64-with-glibc2.17
Using profiles.yml file at /root/.dbt/profiles.yml
Using dbt_project.yml file at /root/spark_demo/spark_demo/dbt_project.yml

Configuration:
  profiles.yml file [OK found and valid]
  dbt_project.yml file [OK found and valid]

Required dependencies:
 - git [OK found]

Connection:
  host: mcdp1
  port: 10000
  cluster: None
  endpoint: None
  schema: dbt_example
  organization: 0
  Connection test: [OK connection ok]

All checks passed!
```

## 总结

不透出具体、友好的错误信息，排查会很困难。但是开源的最大好处是代码中的细节无处可藏，只要看代码，再加上一些调试总能找到具体问题所在。

dbt上手简单，配套生态已很完善，其他的待进一步测试。