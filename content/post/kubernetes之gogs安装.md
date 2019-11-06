---
title: kubernetes之gogs安装
subtitle: 配置LDAP验证
date: 2019-08-26
tags: ["kubernetes", "ldap", "docker","kubectl","yaml","gogs","git"]
keywords: ["kubernetes", "ldap",  "更新", "kubectl", "docker","gogs","devops"]
slug: kubernetesgogs
bigimg: [{src: "/img/great-wall-01.jpg", desc: ""}]
category: "kubernetes"
---

Gogs 是一款极易搭建的自助 Git 服务。
Gogs 的目标是打造一个最简单、最快速和最轻松的方式搭建自助 Git 服务。使用 Go 语言开发使得 Gogs 能够通过独立的二进制分发，并且支持 Go 语言支持的 所有平台，包括 Linux、Mac OS X、Windows 以及 ARM 平台。

本文介绍在kubernetes安装gogs git服务

<!--more-->

## 安装
### 编写pvc.yaml
用于gogs数据持久化存储
```yaml
apiVersion: v1
metadata:
  name: gogs-pvc
  namespace: basics
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: alicloud-disk-available
```
```shell
kubectl apply -f pvc.yaml
[user@S001 basics]# kubectl get pvc
NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS              AGE
gogs-pvc   Bound    pvc-37ecf2e0-6723-4ff8-9af8-72058aa44d3c   5Gi        RWO            alicloud-disk-available   59s
```
* 存储使用了阿里云的nfs，storageClass模式

### 编写gogs-deployment.yaml
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: gogs
  labels:
    app: gogs
  namespace: basics
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gogs
  template:
    metadata:
      name: gogs
      labels:
        app: gogs
    spec:
      containers:
      - name: gogs
        image: gogs/gogs:0.11.86
        imagePullPolicy: IfNotPresent
        env:
        - name: TZ
          value: Asia/Shanghai

        volumeMounts:
          - name: gogs-data
            mountPath: /data
        ports:
        - containerPort: 3000
          name: web
          protocol: TCP
        - containerPort: 22
          name: ssh
          protocol: TCP
        resources:
          limits:
            cpu: "1000m"
            memory: "1024Mi"
          requests:
            cpu: "500m"
            memory: "512Mi"
        livenessProbe:
          httpGet:
            port: 3000
        readinessProbe:
          httpGet:
            port: 3000
      volumes:
      - name: gogs-data
        persistentVolumeClaim:
          claimName: gogs-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: gogs
  labels:
    app: gogs
  namespace: basics
spec:
  selector:
    app: gogs
  ports:
    - name: http
      port: 3000
  type: ClusterIP
```
```shell
kubectl apply -f gogs-deployment.yaml
[user@S001 ~]# kubectl get pods,svc-n default -l app=gogs
NAME                    READY   STATUS    RESTARTS   AGE
gogs-7499894d8d-vcfpb   2/2     Running   0          22s
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/gogs   ClusterIP   10.88.244.169   <none>        3000/TCP   77s
```

### 外部访问gogs服务
kubernetes集群安装了istio，所以使用了istio-ingressgateway作为ingress
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: gogs-gateway
  namespace: default
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http-gogs
      protocol: HTTP
    hosts:
    - "gogs.zb.vip"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: gogs-vs
  namespace: default
spec:
  hosts:
  - "gogs.zb.vip"
  gateways:
  - gogs-gateway
  http:
  - match:
    - uri:
        prefix: /gogs
    route:
    - destination:
        host: gogs
        port: 
          number: 3000
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: gogs
  namespace: default
spec:
  host: gogs
  trafficPolicy:
    tls:
      mode: DISABLEE
```
```shell
[root@ProApiS001 basics]# kubectl apply -f gogs-gw.yaml 
gateway.networking.istio.io/gogs-gateway created
virtualservice.networking.istio.io/gogs-vs created
destinationrule.networking.istio.io/gogs created
```
安装完成后通过DNS解析，或者修改本机的[hosts](C:\Windows\System32\drivers\etc\hosts)文件进行域名解释

### 初始化设置
利用浏览器登陆到gogs上，如果服务刚安装会跳转到初始化页面
![gogs init](https://www.k8sz.com/img/k8s/gogs001.png)
![gogs init](https://www.k8sz.com/img/k8s/gogs002.png)

* 数据库设置，这里使用了SQLite3数据库
* 域名输入自己的域名
* gogs的默认端口为3000
* 应用URL，这里主要是git URL前缀，如：https://github.com/kubernetes/kubernetes.git
*  默认情况下是没有管理员账户，需要设置，4项都要完整填写。填写完成点击”立即完成“

### gogs使用ldap用户登陆
![gogs ldap](https://www.k8sz.com/img/k8s/gogs003.png)
![gogs ldap](https://www.k8sz.com/img/k8s/gogs004.png)
![gogs ldap](https://www.k8sz.com/img/k8s/gogs005.png)

* 点击右上角用户图标下拉箭头，控制面板，认证源管理
* 填写认证名称
* 主机地址和端口，（这里用的是k8s同命名空间下的openldap服务）
* DN和密码是ldap中管理员和密码，[openldap安装请参考](https://www.k8sz.com/post/kubernetesldap)
* 用户搜索基准，用户属性，名字属性，根据图中设置即可
* 勾选相关设置，点击更新认证设置

### 验证ldap登陆
![gogs ldap](https://www.k8sz.com/img/k8s/gogs006.png)
![gogs ldap](https://www.k8sz.com/img/k8s/gogs007.png)
ldap设置成功后，在登陆页面可以看到，本地和设置的认证源

## 后记
gogs和gitlab相比，足够轻便，是小团队和小型创业公司的不二选择。

<!--adsense-self-->
