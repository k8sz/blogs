---
title: kubernetes之LDAP安装
date: 2019-08-16
tags: ["kubernetes", "ldap", "docker","kubectl","yaml"]
keywords: ["kubernetes", "ldap",  "更新", "kubectl", "docker"]
slug: kubernetesldap
gitcomment: false
bigimg: [{src: "/img/great-wall-01.jpg", desc: ""}]
category: "kubernetes"
---

轻型目录访问协议（英文：Lightweight Directory Access Protocol 缩写: LDAP）是一个开放的，中立的，工业标准的应用协议，通过IP协议提供访问控制和维护分布式信息的目录信息。
可以这样讲：市面上只要你能够想像得到的所有工具软件，全部都支持LDAP协议。比如说你公司要安装一个项目管理工具，那么这个工具几乎必然支持LDAP协议，你公司要安装一个bug管理工具，这工具必然也支持LDAP协议，你公司要安装一套软件版本管理工具，这工具也必然支持LDAP协议。LDAP协议的好处就是你公司的所有员工在所有这些工具里共享同一套用户名和密码，来人的时候新增一个用户就能自动访问所有系统，走人的时候一键删除就取消了他对所有系统的访问权限，这就是LDAP。

本文将介绍如何使用Helm在kubernetes 集群上搭建一个OpenLDAP服务。

<!--more-->

## 安装OpenLDAP
### 编写Helm values.yaml
编写我们需要修改的Helm values.yaml文件，将我们需要修改的字段信息增加到文件中，用来替换默认的字段
```yaml
env:
  LDAP_ORGANISATION: "ZB Inc."
  LDAP_DOMAIN: "zb.com"
  LDAP_BACKEND: "hdb"
  LDAP_TLS: "true"
  LDAP_TLS_ENFORCE: "false"
  LDAP_REMOVE_CONFIG_AFTER_SETUP: "true"
```
这里主要是修改了LDAP_DOMAIN(域名)，和LDAP_ORGANISATION（公司名称），可以根据自己的需求进行修改。
因为各个Chart的版本values.yaml有所不同，需要留意一下，我的Chart版本如下：
```shell
[user@S01 /]# helm search  stable/openldap
NAME           	CHART VERSION	APP VERSION	DESCRIPTION                     
stable/openldap	    1.1.0          2.4.47   Community developed LDAP software
```

### 接下来我们使用helm进行openldap进行安装
首先我们先进行repo更新，并查看openldap Chart最新版本
```shell
[user@S01 openldap]# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "incubator" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete.
[user@S01 /]# helm search  stable/openldap
NAME           	CHART VERSION	APP VERSION	DESCRIPTION                     
stable/openldap	    1.1.0          2.4.47   Community developed LDAP software
```
接下来进行openldap安装
```shell
[user@S01 ~]# helm install -n openldap --namespace default -f zb.yaml stable/openldap
NAME:   openldap
LAST DEPLOYED: Fri Aug 16 14:37:24 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME          DATA  AGE
openldap-env  6     0s

==> v1/Pod(related)
NAME                       READY  STATUS             RESTARTS  AGE
openldap-858575df78-6kpgd  0/1    ContainerCreating  0         0s

==> v1/Secret
NAME      TYPE    DATA  AGE
openldap  Opaque  2     0s

==> v1/Service
NAME      TYPE       CLUSTER-IP      EXTERNAL-IP  PORT(S)          AGE
openldap  ClusterIP  10.109.187.113  <none>       389/TCP,636/TCP  0s

==> v1beta2/Deployment
NAME      READY  UP-TO-DATE  AVAILABLE  AGE
openldap  0/1    1           0          0s


NOTES:
OpenLDAP has been installed. You can access the server from within the k8s cluster using:

  openldap.default.svc.cluster.local:389


You can access the LDAP adminPassword and configPassword using:

  kubectl get secret --namespace default openldap -o jsonpath="{.data.LDAP_ADMIN_PASSWORD}" | base64 --decode; echo
  kubectl get secret --namespace default openldap -o jsonpath="{.data.LDAP_CONFIG_PASSWORD}" | base64 --decode; echo


You can access the LDAP service, from within the cluster (or with kubectl port-forward) with a command like (replace password and domain):
  ldapsearch -x -H ldap://openldap-service.default.svc.cluster.local:389 -b dc=example,dc=org -D "cn=admin,dc=example,dc=org" -w $LDAP_ADMIN_PASSWORD


Test server health using Helm test:
  helm test openldap


You can also consider installing the helm chart for phpldapadmin to manage this instance of OpenLDAP, or install Apache Directory Studio, and connect using kubectl port-forward.
```
因admin密码是随机生成并使用base64进行了加密，所以我们可以根据helm安装提示获取openldap的admin密码
```shell
[user@S01 ~]# kubectl get secret --namespace default openldap -o jsonpath="{.data.LDAP_ADMIN_PASSWORD}" | base64 --decode; echo
B74G7RiwwSYXC8LRqAagNaqkDGcZ5RRL
```

### 验证openldap是否安装成功
输入一下命令查看openldap是否安装成功
```shell
[root@ZabbixServer01 ~]# helm list openldap
NAME    	REVISION	UPDATED                 	STATUS  	CHART         	APP VERSION	NAMESPACE
openldap	1       	Fri Aug 16 14:37:24 2019	DEPLOYED	openldap-1.1.0	2.4.47     	default
```
或者使用kubetl查看Pod的状态进行验证
```shell
[root@ZabbixServer01 ~]# kubectl get pods openldap-858575df78-6kpgd 
NAME                        READY   STATUS    RESTARTS   AGE
openldap-858575df78-6kpgd   1/1     Running   0          11m
```
STATUS为Running表示Pod运行正常

### openldap端口开放和客户端连接
默认情况下openladp只能在kubernetes集群上通信，此处使用nodeport的方式进行端口开放
直接修改集群openldap service
```shell
[user@S01 ~]# kubectl get svc openldap 
NAME       TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                   AGE
openldap   NodePort   10.109.187.113   <none>        389:389/TCP,636:636/TCP   50m

[user@S01 ~]# kubectl edit -n default svc openldap

# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2019-08-16T06:37:24Z"
  labels:
    app: openldap
    chart: openldap-1.1.0
    heritage: Tiller
    release: openldap
  name: openldap
  namespace: default
  resourceVersion: "9925276"
  selfLink: /api/v1/namespaces/default/services/openldap
  uid: 4eb64b71-bff0-11e9-97d3-525400eeff5c
spec:
  clusterIP: 10.109.187.113
  ports:
  - name: ldap-port
    port: 389
    nodePort: 389
    protocol: TCP
    targetPort: ldap-port
  - name: ssl-ldap-port
    port: 636
    nodePort: 636
    protocol: TCP
    targetPort: ssl-ldap-port
  selector:
    app: openldap
    release: openldap
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```
添加nodePort: 389 和nodePort: 636 并将type修改为NodePort ，默认情况下kubernetes开放的端口为30000-32768，需要开放其他端口需要修改kube-apiserver启动参数
查看主机端口开放情况
```shell
[user@S01 ~]# netstat -tnpl |grep :389
tcp6       0      0 :::389                  :::*                    LISTEN      5285/kube-proxy     
```
下载客户端：[LdapAdmin](https://gitee.com/k8sz/zb/raw/master/LdapAdmin.exe)
打开客户端进行连接：
![LdapAdmin](https://www.k8sz.com/img/k8s/ldap001.png)

* 点击Start或者图标，弹出Connections页面
* New connections添加一个新连接
* 输入Connection name
* 输入openldap Host IP或者域名（前提要域名能解释到Host IP）
* 如果与Host连接正常，点击Fetch DNs会自动填充Base信息
* 输入Username，如：cn=admin,dc=zb,dc=com  Password为我们安装openldap时admin的密码
* 点击Test connection,所有信息填写正确，会弹出提示框提示Connection is successful.
* 点击OK进行保存使用客户端登陆到openldap后就可以进行OU和用户创建

## 结语
至此我们还有几个地方需要进行优化，最重要的就是数据持久化存储。

接下来，会继续分享第三方应用使用ldap进行登陆

<!--adsense-self-->
