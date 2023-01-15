---
title: "在CentOS 6.4上设置Python 2.7.6和3.3.3环境"
date: "2017-08-26"
draft: false
tags: ["program"]
keywords: ["centos", "python", "安装"]
categories: ["技术"]
---

`写在前面`: 虽然我也认为目前使用Ubuntu用作服务器系统不存在任何问题,但是很多时候基于安全性与稳定性考虑还是难免会选择红帽系列。更多时候作为小码农也没有权力做技术决策吧[无奈状]。话说回来,感觉难受只是因为不熟悉,熟悉也就好了。作为Ubuntu党,这些天一直在与CentOS打交道,一通折腾下来,畅快很多。于是想写篇博客记录,却发现Digital Ocean上的一篇博文写的不能再好了(Digital Ocean上的文章都是精品)。所以,还是直接翻译吧。[原文链接](https://www.digitalocean.com/community/tutorials/how-to-set-up-python-2-7-6-and-3-3-3-on-centos-6-4)。

## 简介

为了让应用在特定的环境中运行,管理服务器常常是作为开发者的责任之一。当面临选择操作系统时(尤其是生产环境),基于安全等方面的考虑,CentOS会是呼声最高的操作系统之一。

然而,当你开始使用CentOS时就会傻眼了,CentOS内置的Python版本还是2.6(甚至2.4.3),这用于应用显然是不合适的。`注:由于Linux系统很多方面都依赖Python,所以一般是不能直接替换版本的`。

这篇文章将会讲述如何下载和设置Python(2.7.6和3.3.3)而不破坏内置的2.6(或2.4)版本的Python。这非常重要,因为一些像YUM等一些系统工具是依赖内置版本的。同时将安装`pip`和`virualenv`这两个非常流行的Python组件。

读完这篇文章,你将能同时使用任意版本的Python,创建虚拟环境以及为任意版本的Python下载和管理开发包。

## CentOS及它的设计理念

开始安装之前,先来说说CentOS。

> 为什么CentOS使用旧版本的应用?

CentOS是源于RHEL(Red Hat Enterprise Linux)的一个社区发行版。而RHEL是面向商业用户的,因此需要保证长期的支持和稳定性。

因此,追求系统的稳定性是最重要的原因,所有的软件都是通过了测试,更加稳定的版本。这背后的哲学就是:`如果没有崩溃,不要去修复。(if it ain't broke,don't fix it)`

> 为什么开发版的库和应用需要另外安装并与内置版本共存?

默认的CentOS版本身并不带有很多的工具,而那些内置的工具往往是系统依赖的(如:YUM)。如果需要任何时候都保持系统的稳定性,不破坏系统的任何东西,在改动他们的时候就需要格外的小心了。

不要认为系统发行版中的工具都可以为你所用而养成按照自己的需要进行设置的习惯。

使用这篇简单易用的说明,你将可以使用任意版本的Python,同时你也能从中学会如何安装其他应用(使用源码包)。

## 系统准备,安装Python

像其他的程序一样,在CentOS中安装Python需要简单的几个步骤:更新系统,获取所需版本的Python,进行设置。

`提示`: 你可以在Python的[发行页面](http://www.python.org/download/releases/)中查看和选择你所需要的版本,使用本教程你可以安装其中任意版本或所有版本。

`注`: 本教程适用于CentOS 6.5、5.8以及6.4版本。

## 更新内置CentOS应用

开始安装之前,确保已经将内置的应用升级到最新的可用版本。
使用以下命令即可进行升级:

```bash
yum -y update
```

## 系统准备

CentOS是简洁的,甚至太简洁了也不为过。这就意味着很多你想使用的应用和工具默认都没有安装。

这是设计哲学决定的,我们需要一些库和工具(比如:development [related] tools)默认都没有在发行版中安装。因此,再我们继续下一步之前需要先下载和安装他们。


使用包管理工具`yum`我们有两种方法在系统中安装这些工具:

**选择1**(不推荐)一个个的下载和安装这些工具(如:make,gcc等)。这样很可能再安装其中一个工具的时候发现它依赖另一个库或者工具,这时会抛出错误,而你不得不回头重新下载和安装。

正确和推荐的方法是**选择2**,使用简单的yum命令一次性安装所有的软件组。

## YUM Software Group

yum的软件组是由一些常用的工具包组成的,只需要知道软件组的名称就能使用一个命令就能同时安装和执行所有的软件。甚至可以一次性同时下载和安装多个软件组。

我们现在所需要的软件组就是`Development Tools`。

## 怎样使用YUM在CentOS中安装Development Tools

使用下面的命令安装所需的development tools:

```bash
yum groupinstall -y development
```

或:

```bash
yum groupinstall -y 'development tools'
```

`注`:一些包在旧版本的CentOS中不能正常工作。

安装一些附加的包:

```bash
yum install -y zlib-dev openssl-devel sqlite-devel bzip2-devel
```

`提示`:以上这些附加的包虽然是可选的,但是在大部分的任务中它们都是很常用的,或者以后会用到。除非你使用其他高级的方法进行了安装,否则Python在编译的过程中将不能链接他们,这样在使用过程中可能会出现一些问题。

## 源码安装Python

在系统中设置Python需要以下3个步骤和4个工具:

- **下载** 下载源码压缩包(wget）
- **解压** 解压缩安装包(tar)
- **设置** 和 **构建** (autoconf(configure)/make)

*GNU wget*

GNU的wget工具可以使用多种协议(HTTP,FTP)下载文件。尽管之前旧版本的CentOS都缺失,但是现在是自带的工具了。

```bash
使用示例: wget [URL]
```

*GNU Tar*

GNU的Tar工具是基本的压缩文件和解压缩的工具.

```bash
使用示例: tar [options] [argments]
```

*GNU autoconf and GNU make*

autoconf和make是两个不同的工具,常一起使用来进行软件的编译与安装。

- **使用./configure**在安装前构建源码
- **使用 make**进行库链接工作
- **使用 make install** 安装软件

## 下载,构建(编译),安装Python

本节将所有的介绍都可以换成你所需的Python版本。完成之后你将能同时使用多个版本的Python,只是有时候你需要明确指出你所使用的版本,如使用python2.7或者python3.3来代替默认的python命令。

#### 下载源码压缩包

使用wget从网上下载所需的安装包,这里使用`2.7.6`版本。

```bash
wget http://www.python.org/ftp/python/2.7.6/Python-2.7.6.tar.xz
```

3.3.3版本:

```bash
wget http://www.python.org/ftp/python/3.3.3/Python-3.3.3.tar.xz
```

(可选步骤):`xz`工具:可以看出这里的压缩包是使用`XZ`库进行压缩的,你的系统中也许不包括这个工具,如果没有可以使用下面的命令进行安装:

```bash
yum install xz-libs
```

#### 解压源码包

这里的解压的包括两个步骤:先解压`xz`包,再解压`tar`包。

```bash
xz -d Python-2.7.6.tar.xz

tar -xvf Python-2.7.6.tar
```

3.3.3版本示例:

```bash
xz -d Python-3.3.3.tar.xz
tar -xvf Python-3.3.3.tar
```

#### 编译与安装

在构建源码之前需要确保所有的依赖和工具都已经准备好了。构建过程是自动的。

```bash
# 进入源码包目录
cd Python-2.7.6

# 开始构建之前指定安装的目录
# 默认会被安装进 /usr/local目录
# 可以使用--prefix参数来进行指定
./configure --prefix=/usr/local
```

3.3.3版本示例:

```bash
cd Python-3.3.3
./configure
```

因为我们已经提前下载了所需的工具和应用,所以这个过程将顺利地自动执行。当完成的时候就可以进入下一步:*构建与安装*

#### 构建与安装

一般我们应该使用`make install`来进行安装,但是为了不覆盖系统默认的版本,我们使用`make altinstall`。

```bash
# 构建源码
# 将持续一段时间
make

# 安装
make altinstall
```

3.3.3版本示例:

```bash
make & make altinstall   # 两个命令可以合为一个
```

#### (可选步骤)将新版本Python目录添加进PATH

> 如果你是完全按照本教程进行操作的,那可以跳过本部分内容。如果你选择了不是/usr/local的目录来安装Python,就需要使用这部分内容来避免每次使用新安装的Python时都输入可执行命令的全路径。

在安装完成之后,我们能够使用全路径来使用可执行的二进制文件,但是只有该版本Python的二进制可执行文件已经添加进了PATH变量之后才能在任意位置使用该可执行文件。

```bash
# example: export PATH="[/path/to/installation]:$PATH"
export PATH="/usr/local/bin:$PATH"
```

## 设置基本的Python组件pip和virtualenv

安装完了Python,现在可以来完成最后的进行部署应用的生产环境了。我们需要设置两个常用的Python组件:`pip`包管理工具和`virtualenv`虚拟环境管理工具。

下面这篇文章详细地介绍了这两个工具[ Common Python Tools: Using virtualenv, Installing with Pip, and Managing Packages](https://www.digitalocean.com/community/articles/common-python-tools-using-virtualenv-installing-with-pip-and-managing-packages)

#### 使用新的Python来安装pip

安装`pip`之前,需要安装它唯一的依赖:`setuptools`

执行以下命令即可安装Pythoon2.7.6的pip:

```bash
# 使用wget下载
wget --no-check-certificate https://pypi.python.org/packages/source/s/setuptools/setuptools-1.4.2.tar.gz

# 解压
tar -xvf setuptools-1.4.2.tar.gz

# 进入目录
cd setuptools-1.4.2

# 使用刚安装的Python安装
python2.7 setup.py install
```

#### 下载pip文件,使用Python2.7进行安装

```bash
curl https://raw.githubusercontent.com/pypa/pip/master/contrib/get-pip.py | python2.7 -
```

#### 安装virtualenv

现在已经安装好了pip,就可以使用它来安装virtualenv了。

执行下面的命令下载和安装virtualenv:

```bash
pip install virtualenv
```

----
以上,就完成了所有的工作。在实际的操作中我发现,目前CentOS6的新版,在编译安装Python时不使用`make altinstall`而使用`make install`不会出现任何问题,这时,使用`python`命令可以进入新安装版本的交互环境,而yum不会出现异常,这倒是个不错的改进。