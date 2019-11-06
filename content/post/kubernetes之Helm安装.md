---
title: kubernetes之Helm安装
date: 2019-08-17
tags: ["kubernetes", "helm", "docker","kubectl","yaml"]
keywords: ["kubernetes", "helm", "kubectl"]
slug: kuberneteshelm
bigimg: [{src: "/img/great-wall-01.jpg", desc: ""}]
category: "kubernetes"
---

Helm可帮助您管理Kubernetes应用程序 - Helm Charts可帮助您定义，安装和升级最复杂的Kubernetes应用程序。Helm就相当于kubernetes环境下的yum包管理工具。

<!--more-->

## Helm结构
### chart
是Helm管理的安装包，里面包含需要部署的安装包资源。类似于yum中的rpm文件。每个Chart包含下面两部分：包的基本描述文件Chart.yaml放在templates目录中的一个或多个Kubernetes manifest文件模板。
### Release
在Kubernetes集群上运行的一个Chart实例。在同一个集群上，一个Chart可以安装多次。例如一个MySQL Chart，如果想在服务器上运行两个MySQL数据库，就可以基本这个Chart安装两次。每次安装都会生成新的Release,会有独立的Release名称。
### Repository
用于存放和共享Chart的仓库。

## Helm组件
Helm 有以下两个组成部分：
Helm Client 是用户命令行工具，其主要负责如下：
helm客户端，管理本地的Chart仓库，管理chart，于Tiller服务器交互，发送Chart，实例安装，查询，卸载等操作
Tiller服务端（k8s node上安装），接收helm发送过来的

## 安装
### helm安装
首先我们需要在[releases](https://github.com/helm/helm/releases)下载对应版本的安装包
```shell
wget https://get.helm.sh/helm-v2.14.3-linux-amd64.tar.gz
tar xf helm-v2.14.3-linux-arm64.tar.gz
cd linux-amd64
cp -a helm /usr/bin
helm version
```

### Tiller安装
建立Tiller ServiceAccount并绑定Cluster-admin角色,Tiller-sa-rbac.yaml
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```
使用kubectl apply
```shell
[user@S001 ~]# kubectl apply -f Tiller-sa-rbac.yaml 
serviceaccount/tiller created
clusterrolebinding.rbac.authorization.k8s.io/tiller created
```

运行helm init 命令进行Tiller安装
```shell
helm init --upgrade --service-account tiller --tiller-image registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.14.3
```
### 验证Tiller
```shell
[user@S01 k8sz]# kubectl get pods -n kube-system -l name=tiller
NAME                             READY   STATUS    RESTARTS   AGE
tiller-deploy-78f54ccfd4-jxdgb   1/1     Running   0          1d
```

### helm常用命令
chart操作
```shell
#查看版本信息
helm version
#更新仓库到本地
helm repo update
#查看helm仓库中的chart
helm search 
#查看chart详细信息
helm inspect stable/jenkins
# 添加 incubator repo
helm repo add incubator https://aliacs-app-catalog.oss-cn-hangzhou.aliyuncs.com/charts-incubator/
# 查询 repo 列表
helm repo list
# 生成 repo 索引（用于搭建 helm repository）
helm repo index
#删除一个repo
helm repo remove incubator
```
release操作
```shell
#部署安装release
helm install --name mem1 stable/tomcat
#升级release
helm upgrade --set mysqlRootPassword=passwd db-mysql stable/mysql
#回滚release
helm rollback db-mysql 1 
#查看Release列表
helm listh	
helm ls 
#删除release
helm delete --purge redis
#测试安装release，并不是实际安装
helm install --dry-run --debug --name els --namespace=efk -f values.yaml incubator/elasticsearch
# 创建一个新的 chart
helm create hello-chart
# validate chart
helm lint
# 打包 chart 到 tgz
helm package hello-chart
#生成根据chart生产kubernetes yaml文件
helm template install/kubernetes/helm/istio-init --name istio-init --namespace istio-system > istio.yaml
```

## 结语
helm给kubernetes操作带来了很多方便，这里提供一个[webui helm](https://kubeapps.com/)

<!--adsense-self-->
