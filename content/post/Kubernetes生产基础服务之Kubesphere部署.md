---
title: Kubernetes生产基础服务之Kubesphere部署
subtitle: Kubernetes生产之基础篇
date: 2019-10-26
tags: ["kubernetes", "concept", "service","pod","ingress","namespace","volume","pv","pvc","deployment","StatefulSet","Job","CronJob","HPA","Service Account","Secret","ConfigMap","Resource Quotas"]
keywords: ["kubernetes", "concept", "service","pod","ingress","namespace","volume","pv","pvc","deployment","StatefulSet","Job","CronJob","HPA","Service Account","Secret","ConfigMap","Resource Quotas"]
slug: kubernetes-kubesphere
gitcomment: false
bigimg: [{src: "https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/photo-1565780421727-543646cefafb.jpeg", desc: ""}]
category: "kubernetes"
---

KubeSphere 是在目前主流容器调度平台 Kubernetes 之上构建的企业级分布式多租户容器平台，提供简单易用的操作界面以及向导式操作方式，在降低用户使用容器调度平台学习成本的同时，极大减轻开发、测试、运维的日常工作的复杂度，旨在解决 Kubernetes 本身存在的存储、网络、安全和易用性等痛点。除此之外，平台已经整合并优化了多个适用于容器场景的功能模块，以完整的解决方案帮助企业轻松应对敏捷开发与自动化运维、微服务治理、多租户管理、工作负载和集群管理、服务与网络管理、应用编排与管理、镜像仓库管理和存储管理等业务场景。

KubeSphere为我们提供了可视化 CI/CD 流水线、多维度监控告警日志、多租户管理、LDAP 集成、新增支持 HPA (水平自动伸缩) 、容器健康检查以及 Secrets、ConfigMaps 的配置管理等功能，新增微服务治理、灰度发布、s2i、代码质量检查等。一定程度上简化了我们管理kubesnetes集群。

<!--more-->

## 在 Kubernetes 在线部署 KubeSphere

### 创建Namespace

在 Kubernetes 集群中创建名为 kubesphere-system 和 kubesphere-monitoring-system 的 namespace
```shell
cat <<EOF | kubectl create -f -
---
apiVersion: v1
kind: Namespace
metadata:
    name: kubesphere-system
---
apiVersion: v1
kind: Namespace
metadata:
    name: kubesphere-monitoring-system
EOF
```

### 创建 Kubernetes 集群 CA 证书的 Secret
```shell
kubectl -n kubesphere-system create secret generic kubesphere-ca --from-file=ca.crt=/etc/kubernetes/pki/ca.crt --from-file=ca.key=/etc/kubernetes/pki/ca.key 
```

### 创建 etcd 的证书 Secret
```shell
kubectl -n kubesphere-monitoring-system create secret generic kube-etcd-client-certs --from-file=etcd-client-ca.crt=/etc/kubernetes/pki/etcd/ca.crt --from-file=etcd-client.crt=/etc/kubernetes/pki/etcd/healthcheck-client.crt --from-file=etcd-client.key=/etc/kubernetes/pki/etcd/healthcheck-client.key
```

### 克隆 kubesphere-installer 仓库至本地
```shell
git clone https://github.com/kubesphere/ks-installer.git
```

### 修改安装配置文件
```shell
cd ks-installer/deploy
vim kubesphere-installer.yaml
```

```yaml
apiVersion: v1
data:
  ks-config.yaml: |
    kube_apiserver_host: 192.168.1.1:6443
    etcd_tls_enable: True
    etcd_endpoint_ips: 192.168.1.1
    disableMultiLogin: False
    istio_enable: False
    keep_log_days: 30
    sonarqube_enable: False
    metrics_server_enable: False
    elk_prefix: logstash
    persistence:
      enable: True
      storageClass: "alicloud-disk-available"
......
```

配置文件关闭了istio，sonarqube，metrics_server之前有安装所以这里不安装，日志保存30天，开启storageClass参考[kubernetes 挂载阿里云nas存储作为StorageClass](https://www.k8sz.com/post/kubrenetes-nas-storageclass),开启账户可以多终端登录
运行部署
```shell
kubectl apply -f kubesphere-installer.yaml
```

### 查看部署日志
```shell
kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l job-name=kubesphere-installer -o jsonpath='{.items[0].metadata.name}') -f
```

### kubesphere镜像无法下载
```shell
#登陆到镜像仓库，每个nodes都需要操作
docker login -u guest -p guest dockerhub.qingcloud.com
```

### 查看控制台的服务端口
```shell
# 查看 ks-console 服务的端口  默认为 NodePort: 30880 默认的集群管理员账号为 admin/P@88w0rd
kubectl get svc -n kubesphere-system
```

## 参考

* https://kubesphere.io/docs/v2.0/zh-CN/installation/install-on-k8s/

<!--adsense-self-->
