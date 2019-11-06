---
title: Kubernetes 开篇
date: 2019-08-15
tags: ["kubernetes", "devops"]
slug: Kubernetesopenings
keywords: ["kubernetes", "技术", "分享", "实践"]
bigimg: [{src: "/img/great-wall-01.jpg", desc: ""}]
category: "kubernetes"
---
Kubernetes是一个可移植，可扩展的开源平台，用于管理容器化工作负载和服务，有助于声明性配置和自动化。作为一个从业运维工作的人员，得益与容器Docker和容器编排Kubernetes的出现，解决了许多运维中出现的痛点，提高运维效率。Kubernetes是如何解决运维中出现的痛点的呢？

<!--more-->

- Kubernetes 主控组件（Master） 包含三个进程，都运行在集群中的某个节上，通常这个节点被称为 master 节点。这些进程包括：kube-apiserver、kube-controller-manager和kube-scheduler。
- 集群中的每个非 master 节点都运行两个进程：
    - kubelet，和 master 节点进行通信。
    - kube-proxy，一种网络代理，将 Kubernetes 的网络服务代理到每个节点上。

## kubernetes优点
日常的运维工作中，我们需要编写足够多的shell脚本和管理配置文件来保证服务器和应用的正常运行，管理过多的shell脚本和配置文件，难免会出现错误，Kubernetes为我们解决了很多问题，这也是Kubernetes的优点，如  服务发现和负载平衡  存储编排  自动部署和回滚 资源限制 自我修复 密钥和配置管理

### 服务发现和负载均衡
Kubernetes可以使用DNS名称或使用自己的IP地址公开容器。如果容器的流量很高，Kubernetes能够负载均衡并分配网络流量，以便部署稳定。

### 存储编排
Kubernetes允许您自动安装您选择的存储系统，例如本地存储，公共云提供商等。

### 自动部署和回滚
您可以使用Kubernetes描述已部署容器的所需状态，并且可以以受控速率将实际状态更改为所需状态。例如，您可以自动化Kubernetes为您的部署创建新容器，删除现有容器并将所有资源用于新容器。

### 资源限制
Kubernetes允许您指定每个容器需要多少CPU和内存（RAM）。当容器指定了资源请求时，Kubernetes可以更好地决定管理容器的资源。

### 自我修复
Kubernetes重新启动失败的容器，替换容器，杀死不响应用户定义的运行状况检查的容器，并且在它们准备好服务之前不会将它们通告给客户端。

### 密钥和配置管理
Kubernetes允许您存储和管理敏感信息，例如密码，OAuth令牌和ssh密钥。您可以部署和更新机密和应用程序配置，而无需重建容器映像，也不会在堆栈配置中暴露机密

## Kubernetes组件
### kube-apiserver
暴露Kubernetes API的master上的组件。它是Kubernetes控制飞机的前端。
它旨在水平扩展 - 也就是说，它通过部署更多实例来扩展

### ETCD
一致且高度可用的键值存储，用作Kubernetes的所有群集数据的后备存储。

### kube-scheduler
主服务器上的组件，用于调度新创建pod和监控节点，并选择一个节点供其运行。
调度决策所考虑的因素包括个人和集体资源需求，硬件/软件/策略约束，亲和力和反亲和性规范，数据位置，工作负载和最后期限。

### kube-controller-manager
运行控制器的主服务器上的组件。
逻辑上，每个控制器是一个单独的过程，但为了降低复杂性，它们都被编译成单个二进制文件并在单个进程中运行。
这些控制器包括：

- 节点控制器：负责在节点出现故障时注意和响应。
- 复制控制器：负责为系统中的每个复制控制器对象维护正确数量的pod。
- 端点控制器：填充端点对象（即，连接服务和窗格）。
- 服务帐户和令牌控制器：为新的命名空间创建默认帐户和API访问令牌。

## 节点组件
节点组件在每个节点上运行，维护正在运行的pod并提供Kubernetes运行时环境。

### kubelet
在群集中的每个节点上运行的代理。它确保容器在pod中运行。
kubelet采用一系列通过各种机制提供的PodSpecs，并确保这些PodSpecs中描述的容器运行正常。kubelet不管理不是由Kubernetes创建的容器。

### kube-proxy
kube-proxy 是在群集中的每个节点上运行的网络代理。
它通过维护主机上的网络规则并执行连接转发来实现Kubernetes服务抽象。
kube-proxy负责请求转发。kube-proxy允许通过一组后端功能进行TCP和UDP流转发或循环TCP和UDP转发。

### Container Runtime
容器运行时是负责运行容器的软件。
Kubernetes支持多个容器运行时：[Docker](http://www.docker.com/)， [containerd](https://containerd.io/)，[cri-o](https://cri-o.io/)， [rktlet](https://github.com/kubernetes-incubator/rktlet)以及[Kubernetes CRI（容器运行时接口）的](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/container-runtime-interface.md)任何实现。

<!--adsense-self-->
