---
title: kubernetes之Jenkins x安装
date: 2019-09-16
tags: ["kubernetes", "ldap", "docker","kubectl","yaml","jenkins","ci/cd","jx"]
keywords: ["kubernetes", "ldap",  "更新", "kubectl", "docker"]
slug: kubernetes-jx-install
gitcomment: true
bigimg: [{src: "/img/k8s/k8s001.jpg", desc: ""}]
category: "kubernetes"
---

jx是云原生CICD，devops的一个最佳实践之一，目前在快速的发展成熟中。

本文主要介绍如何在现有kuberneter 上安装Jenkins x，

<!--more-->

## 安装最新版GIT
### 原因
jx 安装要求git版本最低位2.15.0，而在centos7.5使用yum源安装的git版本为1.8.3，git版本低，会出现jx安装过程中使用到的githun 仓库无法正常clone。

### 安装依赖
```shell
yum install curl-devel expat-devel gettext-devel openssl-devel zlib-devel 
yum install gcc perl-ExtUtils-MakeMaker
```

### 下载git源码包
```shell
wget https://mirrors.edge.kernel.org/pub/software/scm/git/git-2.23.0.tar.xz
tar xf git-2.23.0.tar.xz
cd git-2.23.0
```

### 编译安装
```shell
./configure --without-iconv
make CFLAGS=-liconv prefix=/usr/local/git all
make CFLAGS=-liconv prefix=/usr/local/git install
或者
./configure
make && make install
```

### 创建软连接
```shell
cd /usr/bin
rm -f git
ln -s  /usr/local/git/bin/git git
```

### 验证安装版本
```shell
git --version
```

## 安装Jenkins x
### 下载jx可执行二进制文件压缩包
```shell
wget https://github.com/jenkins-x/jx/releases/download/v2.0.696/jx-linux-amd64.tar.gz
tar xzv jx-linux-amd64.tar.gz  -C ~/.jx/bin
export PATH=$PATH:~/.jx/bin
echo 'export PATH=$PATH:~/.jx/bin' >> ~/.bashrc
```
### 建立jx所需要的pv
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkinsx-pv
  labels:
    app: jenkinsx-pv
spec:
  hostPath:
    path: /wd/jenkins
    type: DirectoryOrCreate
  accessModes: ["ReadWriteMany","ReadWriteOnce"]
  capacity:
    storage: 30Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: chartmuseum-pv
  labels:
    app: chartmuseum-pv
spec:
  hostPath:
    path: /wd/chartmuseum
    type: DirectoryOrCreate
  accessModes: ["ReadWriteMany","ReadWriteOnce"]
  capacity:
    storage: 8Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: docker-registry-pv
  labels:
    app: docker-registry-pv
spec:
  hostPath:
    path: /wd/docker-registry
    type: DirectoryOrCreate
  accessModes: ["ReadWriteMany","ReadWriteOnce"]
  capacity:
    storage: 100Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nexus-pv
  labels:
    app: nexus-pv
spec:
  hostPath:
    path: /wd/nexus
    type: DirectoryOrCreate
  accessModes: ["ReadWriteMany","ReadWriteOnce"]
  capacity:
    storage: 8Gi
```
这里使用了本地磁盘做的pv，有条件可以使用storage class

### pull jx所需的docker镜像
shell 脚本
```shell
#!/bin/bash
set -e
K8S_GCR_URL=k8s.gcr.io
GCR_URL=gcr.io
DOCKER=/usr/bin/docker
ALIYUN_URL=mirrorgooglecontainers
MY_URL=gcr.azk8s.cn
MAIN_VERSION=0.1.706
JENKINSX_VERSION=0.0.76
NEXUS_VERSION=0.1.7
JX_VERSION=2.0.645
MAIN_IMAGES=(
builder-maven:$MAIN_VERSION
builder-jx:$MAIN_VERSION
jenkinsx:$JENKINSX_VERSION
jx:$JX_VERSION
nexus:$NEXUS_VERSION
)
for IMAGESNAME in ${MAIN_IMAGES[@]}
do
    $DOCKER pull $MY_URL/$IMAGESNAME
    $DOCKER tag $MY_URL/$MAIN_IMAGESNAME $GCR_URL/$IMAGESNAME
    $DOCKER rmi $MY_URL/$IMAGESNAME
done
```


### 安装
```shell
jx install --external-ip=10.0.0.88 --namespace='jx' --git-provider-url='https://github.com' --git-username='$gitname' --git-api-token='$gittoken' --provider=kubernetes --domain='$domain' -no-default-environments --static-jenkins
```
根据自己的信息填写所需要gitname ，gittoken ，domain

### 验证安装
```shell
[user@S01 h5web]# kubectl get pods 
NAME                                           READY   STATUS      RESTARTS   AGE
jenkins-64b6bc7589-6bx2l                       1/1     Running     1          3d22h
jenkins-x-chartmuseum-d87cbb789-qsvl9          1/1     Running     1          3d22h
jenkins-x-controllerrole-5bc8cc8777-hr685      1/1     Running     1          3d22h
jenkins-x-controllerteam-58565889-5jfvw        1/1     Running     2          3d22h
jenkins-x-controllerworkflow-f64cfb8bb-txp2l   1/1     Running     1          3d22h
jenkins-x-docker-registry-69d666d455-s4cx4     1/1     Running     1          3d22h
jenkins-x-gcactivities-1568613600-94j9v        0/1     Completed   0          98m
jenkins-x-gcactivities-1568615400-bvnt8        0/1     Completed   0          68m
jenkins-x-gcpods-1568613600-82gmq              0/1     Completed   0          98m
jenkins-x-gcpods-1568615400-2nx6p              0/1     Completed   0          68m
jenkins-x-gcpreviews-1568613600-mrsmm          0/1     Completed   0          98m
jenkins-x-heapster-ff6df6848-jbjdc             2/2     Running     2          3d22h
jenkins-x-nexus-6bc788447f-l66zw               1/1     Running     1          3d22h
```

## 参考
https://jenkins-x.io/getting-started/install-on-cluster/


<!--adsense-self-->
