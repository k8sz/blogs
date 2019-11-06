---
title: Linux之常用命令
subtitle: Linux基础篇
date: 2019-10-22
tags: ["linux", "command"]
keywords: ["linux","command"]
slug: linux-command
gitcomment: false
bigimg: [{src: "/img/k8s/k8s001.jpg", desc: ""}]
category: "linux"
---

Linux这么多命令，通常会让初学者望而生畏。
内容：

* ✔ 目录操作
* ✔ 文本处理
* ✔ 压缩
* ✔ 日常运维
* ✔ 系统状态概览
* ✔ 工作常用

<!--more-->

## 目录操作
### 基本操作
工作中，最常打交道的就是对目录和文件的操作。linux提供了相应的命令去操作他，并将这些命令抽象、缩写。
mkdir 创建目录  make dir
cp 拷贝文件  copy
mv 移动文件   move
rm  删除文件 remove
例子：
```shell
# 创建目录和父目录a,b,c,d
mkdir -p a/b/c/d
# 拷贝文件夹a到/tmp目录
cp -rvf a/ /tmp/
# 移动文件a到/tmp目录，并重命名为b
mv -vf a /tmp/b
# 删除机器上的所有文件
rm -rvf /
```
### 目录漫游
ls  命令能够看到当前目录的所有内容。ls -l能够看到更多信息，判断你是谁。
pwd  命令能够看到当前终端所在的目录。告诉你你在哪。
cd  假如你去错了地方，cd命令能够切换到对的目录。
find  find命令通过筛选一些条件，能够找到已经被遗忘的文件
```shell
# 查看/tmp目录
ls /tmp
# 查看当前所在目录
pwd
# 切换到/tmp目录
cd /tmp
# 在/tmp目录下查找文件
find /tmp -name file
```

## 文本处理
### 查看文件
#### cat 
最常用的就是cat命令了，注意，如果文件很大的话，cat命令的输出结果会疯狂在终端上输出，可以多次按ctrl+c终止。
```shell
# 查看文件大小
du -h file
# 查看文件内容
cat file
```
#### less
既然cat有这个问题，针对比较大的文件，我们就可以使用less命令打开某个文件。
类似vim，less可以在输入/后进入查找模式，然后按n(N)向下(上)查找。
有许多操作，都和vim类似，你可以类比看下
#### tail
大多数做服务端开发的同学，都了解这么命令。比如，查看nginx的滚动日志。
```shell
tail -f access.log
```
tail命令可以静态的查看某个文件的最后n行，与之对应的，head命令查看文件头n行。但head没有滚动功能，就像尾巴是往外长的，不会反着往里长。
```shell
tail -n100 access.log
head -n100 access.log
```
### 统计
sort和uniq经常配对使用。
sort可以使用-t指定分隔符，使用-k指定要排序的列
下面这个命令输出nginx日志的ip和每个ip的pv，pv最高的前10
```shell
#2019-06-26T10:01:57+08:00|nginx001.server.ops.pro.dc|100.116.222.80|10.31.150.232:41021|0.014|0.011|0.000|200|200|273|-|/visit|sign=91CD1988CE8B313B8A0454A4BBE930DF|-|-|http|POST|112.4.238.213
awk -F"|" '{print $3}' access.log | sort | uniq -c | sort -nk1 -r | head -n10
```
### 其他
#### grep
grep用来对内容进行过滤，带上--color参数，可以在支持的终端可以打印彩色，参数n则输出具体的行数，用来快速定位。
比如：查看nginx日志中的POST请求
```shell
grep -rn --color POST access.log
```
推荐每次都使用这样的参数。
如果我想要看某个异常前后相关的内容，就可以使用ABC参数。它们是几个单词的缩写，经常被使用。
A  after  内容后n行
B  before  内容前n行
C  count?  内容前后n行
就像是这样:
```shell
grep -rn --color Exception -A10 -B2   error.log
```
#### diff
diff命令用来比较两个文件是否的差异。当然，在ide中都提供了这个功能，diff只是命令行下的原始折衷。对了，diff和patch还是一些平台源码的打补丁方式，你要是不用，就pass吧。

## 压缩
为了减小传输文件的大小，一般都开启压缩。linux下常见的压缩文件有tar、bzip2、zip、rar等，7z这种用的相对较少。
.tar  使用tar命令压缩或解压
.bz2 使用bzip2命令操作
.gz 使用gzip命令操作
.zip 使用unzip命令解压
.rar 使用unrar命令解压
最常用的就是.tar.gz文件格式了。其实是经过了tar打包后，再使用gzip压缩。
### 创建压缩文件
```shell
tar cvfz  archive.tar.gz dir/
```
### 解压
```shell
tar xvfz. archive.tar.gz
```

## 日常运维
### mount
mount命令可以挂在一些外接设备，比如u盘，比如iso，比如刚申请的ssd。可以放心的看小电影了
```shell
mount /dev/sdb1 /xiaodianying
```
### chown
chown 用来改变文件的所属用户和所属组。
chmod 用来改变文件的访问权限。
这两个命令，都和linux的文件权限777有关。
示例：
```shell
# 毁灭性的命令
chmod 000 -R /
# 修改a目录的用户和组为 xjj
chown -R xjj:xjj a
# 给a.sh文件增加执行权限（这个太常用了)
chmod a+x a.sh
```
### yum
假定你用的是centos，则包管理工具就是yum。如果你的系统没有wget命令，就可以使用如下命令进行安装。
```shell
yum install wget -y
```
### systemctl
当然，centos管理后台服务也有一些套路。service命令就是。systemctl兼容了service命令，我们看一下怎么重启mysql服务。 推荐用下面这个。
```shell
service mysql restart
systemctl restart  mysqld
```
对于普通的进程，就要使用kill命令进行更加详细的控制了。kill命令有很多信号，如果你在用kill -9，你一定想要了解kill -15以及kill -3的区别和用途。
### su
su用来切换用户。比如你现在是root，想要用xjj用户做一些勾当，就可以使用su切换。
```shell
su tomcat
su - tomcat
sudo su -
```



<!--adsense-self-->