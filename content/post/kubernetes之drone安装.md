---
title: kubernetes CI/CD之drone安装
subtitle: 基于gogs
date: 2019-08-27
tags: ["kubernetes", "ldap", "docker","kubectl","yaml","gogs","devops","ci/cd","drone"]
keywords: ["kubernetes", "ldap",  "更新", "kubectl", "docker"]
slug: kubernetes-drone-install
gitcomment: false
bigimg: [{src: "/img/great-wall-01.jpg", desc: ""}]
category: "kubernetes"
---

Drone是一种基于容器技术的持续交付系统。使用简单的YAML配置文件来定义和执行Docker容器中定义的Pipeline，Drone主要由两个部分组成：

* Server端负责身份认证，仓库配置，用户，Secrets以及Webhook相关配置
* Agent端端用于接受构建的作业和真正用于运行的 Pipeline 工作流

Server 和 Agent 都是非常轻量级的服务，大概只使用 10~15MB 内存

本文将介绍在kubernetes 集群基于Drone搭建一个CI/CD服务。

<!--more-->

## 安装
### gogs安装
参考[kubernetes之gogs安装](https://k8sz.gitee.io/post/kubernetesgogs/)

### Drone安装
编写drone-pvc.yaml文件
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: dronepvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: alicloud-disk-available
```
在kubernet集群运行drone-pvc.yaml文件建立drone所需的pvc
```shell
kubectl apply -f drone-pvc.yaml
[user@S001 basics]# kubectl get pvc
NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS              AGE
dronepvc   Bound    pvc-7a832ccb-2ee2-4a7b-8cf0-1036357d95b4   5Gi        RWO            alicloud-disk-available   21h
```

编写drone-rbac.yaml文件,针对drone server端和agent端的rbac的授权
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: drone-pipeline
  namespace: default
  labels:
    app: drone
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: drone
  labels:
    app: drone
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: drone-pipeline
  labels:
    app: drone
rules:
  - apiGroups:
      - extensions
    resources:
      - deployments
    verbs:
      - get
      - list
      - watch
      - patch
      - update
  - apiGroups:
      - ""
    resources:
      - namespaces
      - configmaps
      - secrets
      - pods
      - services
    verbs:
      - create
      - delete
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - pods/log
    verbs:
      - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: drone-pipeline
  labels:
    app: drone
subjects:
  - kind: ServiceAccount
    name: drone-pipeline
    namespace: default
roleRef:
  kind: ClusterRole
  name: drone-pipeline
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: drone
  labels:
    app: drone
rules:
  - apiGroups:
      - batch
    resources:
      - jobs
    verbs:
      - "*"
  - apiGroups:
      - extensions
    resources:
      - deployments
    verbs:
      - get
      - list
      - patch
      - update
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: drone
  labels:
    app: drone
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: drone
subjects:
- kind: ServiceAccount
  name: drone
  namespace: default
```
在kubernet集群运行drone-rbac.yaml进行授权
```shell
kubectl apply -f drone-rbac.yaml
```
编写drone-deployment.yaml文件
```yaml
apiVersion: v1
kind: Service
metadata:
  name: drone
  annotations:
  labels:
    app: drone
spec:
  type: ClusterIP
  ports:
  - name: http
    port: 80
    targetPort: 80
  selector:
    app: drone
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: drone-server
  labels:
    app: drone
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: drone
    spec:
      serviceAccountName: drone
      containers:
      - name: server
        image: "docker.io/drone/drone:1.2"
        imagePullPolicy: IfNotPresent
        env:
          - name: DRONE_PRIVATE_MODE
            value: "true"
          - name: DRONE_TLS_AUTOCERT
            value: "true"
          - name: DRONE_KUBERNETES_ENABLED
            value: "true"
          - name: DRONE_KUBERNETES_NAMESPACE
            value: default
          - name: DRONE_GOGS_SERVER
            value: "http://gogs.basics:3000"
          - name: DRONE_GIT_ALWAYS_AUTH
            value: "false"
          - name: DRONE_SERVER_HOST
            value: "drone"
          - name: DRONE_RPC_PROTO
            value: "http"
          - name: DRONE_RPC_HOST
            value: drone.default:80
          - name: DRONE_SERVER_PROTO
            value: http
          - name: DRONE_RPC_SECRET
            value: "b0a7e0969f6736a09a73c1e4933809de"
          - name: DRONE_DATABASE_DATASOURCE
            value: "/var/lib/drone/drone.sqlite"
          - name: DRONE_DATABASE_DRIVER
            value: "sqlite3"
          - name: DRONE_LOGS_DEBUG
            value: "false"
          - name: DRONE_USER_CREATE
            value: username:root,admin:true
        ports:
        - name: http
          containerPort: 80
          protocol: TCP
        - name: https
          containerPort: 443
          protocol: TCP
        - name: grpc
          containerPort: 9000
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /
            port: http       
        volumeMounts:
        - name: data
          mountPath: /var/lib/drone
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: dronepvc
```
这里需要注意POD的参数配置，详情请参考[官方文档](https://readme.drone.io/reference/)
在kubernet集群运行drone-deployment.yaml进行部署
```shell
kubectl apply -f drone-deployment.yaml
[user@S001 ~]# kubectl get pods -l app=drone
NAME                           READY   STATUS    RESTARTS   AGE
drone-server-5f5888d48-7t5lr   1/1     Running   0          10s
```
编写istio ingressgatewqy
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: drone-gateway
  namespace: default
spec:
  selector:
    istio: ingressgateway
  servers:
  - port: 
      number: 80
      name: http-drone
      protocol: HTTP
    hosts:
    - "drone.zb.vip"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: drone-vs
  namespace: default
spec:
  hosts:
  - "drone.zb.vip"
  gateways:
  - drone-gateway
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: drone
        port: 
          number: 80
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: drone
  namespace: default
spec:
  host: drone
  trafficPolicy:
    tls:
      mode: DISABLE
```
在kubernetes集群中运行
```shell
kubectl apply -f drone-gw.yaml
```
最后在DNS解释添加drone.zb.com,并在浏览器中访问drone.zb.com
![drone](https://www.k8sz.com/img/k8s/drone001.png)
</script>
登陆到drone，可以自动或者手动同步gogs用户下的git仓库，选中需要操作的git仓库，点击ACTIVATE REPOSITORY激活

## 结语




<!--adsense-self-->
