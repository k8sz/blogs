---
title: kubernetes之prometheus简介
subtitle: kubernetes监控篇
date: 2019-08-28
tags: ["kubernetes", "cicd","devops","prometheus","监控"]
keywords: ["kubernetes", "ldap",  "更新", "kubectl", "docker"]
slug: kubernetes-prometheus-intro
gitcomment: false
bigimg: [{src: "/img/great-wall-01.jpg", desc: ""}]
category: "kubernetes"
---

kubernetes作为当下最炙手可热的容器编排平台，在给应用部署运维带来便捷的同时，也给应用及性能监控带来了新的挑战。

本文给大家分享一款十分火热的开源监控工具Prometheus，让我们一起来看它是如何兼顾传统的应用监控、主机性能监控和Kubernetes监控的。

<!--more-->

## Prometheus简介
### 什么是Prometheus？

* Prometheus是由SoundCloud使用Go语言开发
* 它是开源监控报警系统和时序列数据库(TSDB)
* 它是Google BorgMon监控系统的开源版本
* 2016年由Google发起Linux基金会旗下的原生云基金会(Cloud Native Computing Foundation), 将Prometheus纳入其下第二大开源项目。Prometheus目前在开源社区相当活跃
* Prometheus和Heapster(Heapster是K8S的一个子项目，用于获取集群的性能数据。)相比功能更完善、更全面。Prometheus性能也足够支撑上万台规模的集群
* Prometheus是一个开源的网络监控工具，它专为监控时间序列数据而构建。你可以按时间长度标准或关键词对来标识时间序列数据。时间序列数据存储在本地磁盘上，以便在紧急情况下轻松访问
* Prometheus的Alertmanager负责消息通知，Alertmanager可以通过电子邮件，PagerDuty或OpsGenie发送通知，如有必要，你也可以关闭警报通知
* Prometheus的UI元素非常出色，允许你从浏览器切换到模板语言和Grafana集成。你还可以将各种第三方数据源从Docker，StatsD和JMX中集成到Prometheus中，来自定义Prometheus
* Prometheus是一个开源的系统监控及告警工具，最初建设在SoundCloud。从2012 Prometheus推出以来，许多公司都采用它搭建监控及告警系统。同时，项目拥有非常活跃的开发者和用户社区
* 它现在是一个独立于任何公司的开源项目，为了强调这一点并明确项目的管理结构，在2016年Prometheus加入CNCF基金会成为继Kubernetes之后的第二个托管项目

### Prometheus有什么特点？

* 多维的数据模型（基于时间序列的k/v键值对）
* 灵活的查询及聚合语句（PromQL）
* 不依赖分布式存储，节点自治
* 基于HTTP的pull模式采集时间序列数据
* 可以使用pushgateway（prometheus的可选中间件）实现push模式
* 可以使用动态服务发现或静态配置采集的目标机器
* 支持多种图形及仪表盘

### Prometheus组件

* Prometheus Server 负责监控数据收集和存储
* Prometheus Alert manager 负责根据告警规则进行告警，可集成很多告警通道
* node-exporter 的作用就是从机器读取指标，然后暴露一个 http 服务，Prometheus 就是从这个服务中收集监控指标。当然 Prometheus 官方还有各种各样的 exporter
* 钉钉集成 Prometheus 告警的组件：prometheus-webhook-dingtalk

### Prometheus适用场景

* 在选择Prometheus作为监控工具前，要明确它的适用范围，以及不适用的场景。
* Prometheus在记录纯数值时间序列方面表现非常好。它既适用于以服务器为中心的监控，也适用于高动态的面向服务架构的监控。
* 在微服务的监控上，Prometheus对多维度数据采集及查询的支持也是特殊的优势。
* Prometheus更强调可靠性，即使在故障的情况下也能查看系统的统计信息。权衡利弊，以可能丢失少量数据为代价确保整个系统的可用性。因此，它不适用于对数据准确率要求100%的系统，比如实时计费系统（涉及到钱）

## Prometheus构架图
![Prometheus构架图](https://www.k8sz.com/img/k8s/prom001.png)

### Prometheus组件
Prometheus生态系统由多个组件组成，它们中的一些是可选的。多数Prometheus组件是Go语言写的，这使得这些组件很容易编译和部署。

* Prometheus Server：Prometheus的核心，根据配置完成数据采集，  服务发现以及数据存储 ,提供PromQL查询语言的支持
*  Push Gateway：为应对部分push场景提供的插件，监控数据先推送到pushgateway上，然后再由server端采集pull。（若server采集间隔期间，pushgateway上的数据没有变化，server将采集2次相同数据，仅时间戳不同）。支持临时性Job主动推送指标的中间网关。
*  Prometheus targets：探针（exporter）提供采集接口，或应用本身提供的支持Prometheus数据模型的采集接口。
*   Service discovery：支持根据配置file_sd监控本地配置文件的方式实现服务发现（需配合其他工具修改本地配置文件），同时支持配置监听Kubernetes的API来动态发现服务。
*  Alertmanager,警告管理器，用来进行报警。
*  PromDash,使用Rails开发可视化的Dashboard，用于可视化指标数据。
*  客户端SDK,官方提供的客户端类库有go、java、scala、python、ruby，其他还有很多第三方开发的类库，支持nodejs、php、erlang等。
*  Exporter是Prometheus的一类数据采集组件的总称。它负责从目标处搜集数据，并将其转化为Prometheus支持的格式。与传统的数据采集组件不同的是，它并不向中央服务器发送数据，而是等待中央服务器主动前来抓取。
*  prometheus_cli,命令行工具。
*  其他辅助性工具,多种导出工具，可以支持Prometheus存储数据转化为HAProxy、StatsD、Graphite等工具所需要的数据存储格式

## Prometheus架构详解
prometheus架构中各个组件是如何协同工作来完成监控任务

### Prometheus server and targets
![Prometheus server and targets](https://www.k8sz.com/img/k8s/prom002.png)
利用Prometheus官方或第三方提供的探针，基本可以完成对所有常用中间件或第三方工具的监控。
![Prometheus](https://www.k8sz.com/img/k8s/prom003.png)
之前讲到Prometheus是中心化的数据采集分析，那这里的探针（exporter）是做什么工作呢？
上图中硬件及系统监控探针node exporter通过getMemInfo()方法获取机器的内存信息，然后将机器总内存数据对应上指标node_memory_MemTotal。

* Jenkins探针Jenkins Exporter通过访问Jenkins的api获取到Jenkins的job数量并对应指标Jenkins_job_count_value。
* 探针的作用就是通过调用应用或系统接口的方式采集监控数据并对应成指标返回给prometheus server。（探针不一定要和监控的应用部署在一台机器） 
* 总的来说Prometheus数据采集流程就是，在Prometheus server中配置探针暴露的端口地址以及采集的间隔时间，Prometheus按配置的时间间隔通过http的方式去访问探针，这时探针通过调用接口的方式获取监控数据并对应指标返回给Prometheus server进行存储。（若探针在Prometheus配置的采集间隔时间内没有完成采集数据，这部分数据就会丢失）

### Prometheus alerting
![Prometheus alerting](https://www.k8sz.com/img/k8s/prom004.png)
Prometheus serve又是如何根据采集到的监控数据配和alertmanager完成告警呢？

* 举一个常见的告警示例，在主机可用内存低于总内存的20%时发送告警。我们可以根据Prometheus server采集的主机性能指标配置这样一条规则node_memory_Active/node_memory_MemTotal < 0.2，Prometheus server分析采集到的数据，当满足该条件时，发送告警信息到alertmanager，alertmanager根据本地配置处理告警信息并发送到第三方工具由相关的负责人接收。
* Prometheus server在这里主要负责根据告警规则分析数据并发送告警信息到alertmanager，alertmanager则是根据配置处理告警信息并发送。

### Alertmanager又有哪些处理告警信息的方式呢

* 分组：将监控目标相同的告警进行分组。如发生停电，收到的应该是单一信息，信息中包含所有受影响宕机的机器，而不是针对每台宕机的机器都发送一条告警信息
* 抑制：抑制是指当告警发出后，停止发送由此告警引发的其他告警的机制。如机器网络不可达，就不再发送因网络问题造成的其他告警。
* 沉默：根据定义的规则过滤告警信息，匹配的告警信息不会发送。

### Service discovery
![Service discovery](https://www.k8sz.com/img/k8s/prom005.png)

* Prometheus支持多种服务发现的方式，这里主要介绍架构图中提到的file_sd的方式。之前提到Prometheus server的数据采集配置都是通过配置文件，那服务发现该怎么做？总不能每次要添加采集目标还要修改配置文件并重启服务吧。
* 这里使用file_sd_configs指定定义了采集目标的文件。Prometheus server会动态检测该配置文件的变化来更新采集目标信息。现在只要能更新这个配置文件就能动态的修改采集目标的配置了。
* 这里采用consul+consul template的方式。在新增或减少探针（增减采集目标）时在consul更新k/v，如新增一个探针，添加如下记录Prometheus/linux/node/10.15.15.132=10.15.15.132:9100，然后配置consul template监控consul的Prometheus/linux/node/目录下k/v的变化，根据k/v的值以及提前定义的discovery.ctmpl模板动态生成Prometheus server的配置文件discovery.yml。

### Web UI
![Web UI](https://www.k8sz.com/img/k8s/prom006.png)
至此，已经完成了数据采集和告警配置，是时候通过页面展示一波成果了。
Grafana已经对Prometheus做了很好的支撑，在Grafana中添加Prometheus数据源，然后就可以使用PromQL查询语句结合grafana强大的图形化能力来配置我们的性能监控页面了。

### 联邦模式
![联邦模式](https://www.k8sz.com/img/k8s/prom007.png)

* 中心化的数据采集存储，分析，而且还不支持集群模式。带来的性能问题显而易见。Prometheus给出了一种联邦的部署方式，就是Prometheus server可以从其他的Prometheus server采集数据。
可能有人会问，这样最后的数据不是还是要全部汇集到Prometheus的global节点吗？
并不是这样的，我们可以在shard节点就完成分析处理，然后global节点直接采集分析处理过的数据进行展示。
* 比如在shard节点定义指标可用内存占比job:memory_available:proportion的结果为(node_memory_MemFree + node_memory_Buffers + node_memory_Cached)/node_memory_MemTotal，这样在shard节点就可以完成聚合操作，然后global节点直接采集处理过的数据就可以了，而不用采集零散的如node_memory_MemFree这类指标。

## Prometheus监控Kubernetes
![Prometheus监控Kubernetes](https://www.k8sz.com/img/k8s/prom008.png)

* Kubernetes官方之前推荐了一种性能监控的解决方案，heapster+influxdb，heapster根据定义的间隔时间从Advisor中获取的关于pod及container的性能数据并存储到时间序列数据库influxdb。
* 也可以使用grafana配置influxdb的数据源并配置dashboard来做展现。而且Kubernetes中pod的自动伸缩的功能（Horizontal Pod Autoscaling）也是基于heapster，默认支持根据cpu的指标做动态伸缩，也可以自定义扩展使用其他指标。
* 但是Heapster无法做Kubernetes下应用的监控。现在，Heapster作为Kubernetes下的开源监控解决方案已经被其弃用，Prometheus成为Kubernetes官方推荐的监控解决方案。
* Prometheus同样通过Kubernetes的cAdvisor接口（/api/v1/nodes/${1}/proxy/metrics/cadvisor）获取pod和container的性能监控数据，同时可以使用Kubernetes的Kube-state-metrics插件来获取集群上Pod, DaemonSet, Deployment, Job, CronJob等各种资源对象的状态，这反应了使用这些资源的应用的状态
* 同时通过Kubernetes api获取node，service，pod，endpoints，ingress等服务的信息，然后通过匹配注解中的值来获取采集目标
![prometheus](https://www.k8sz.com/img/k8s/prom009.png)

* 上面提到了Prometheus可以通过Kubernetes的api接口实现服务发现，并将匹配定义了annotation参数的pod，service等配置成采集目标。那现在要解决的问题就是探针到应用部署配置问题了。
* 这里我们使用了Kubernetes的pod部署的sidecar模式，单个应用pod部署2个容器，利用单个pod中仅共享网络的namespace的隔离特性，探针与应用一同运行，并可以使用localhost直接访问应用的端口，而在pod的注解中仅暴露探针的端口（prometheus.io/port: “9104”）即可。
* Prometheus server根据配置匹配定义注解prometheus.io/scrape: “true”的pod，并将pod ip和注解中定义的端口（prometheus.io/port: “9104”）和路径（prometheus.io/path: “/metrics”）拼接成采集目标http://10.244.3.123:9104/metrics。通过这种方式就可以完成动态添加需要采集的应用。

## 结语

* 参考：https://www.liangzl.com/get-article-detail-8844.html

<!--adsense-self-->
