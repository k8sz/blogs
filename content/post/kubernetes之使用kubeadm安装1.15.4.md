---
title: kubernetes之使用kubeadm安装1.15.4
date: 2019-08-17
tags: ["kubernetes","kubectl","yaml","kubeadm"]
keywords: ["kubernetes", "kubeadm",  "更新", "kubectl", "docker"]
slug: kuberneteskubeadm
gitcomment: true
bigimg: [{src: "/img/great-wall-01.jpg", desc: ""}]
category: "kubernetes"
---

Kubeadm 是一个工具，它提供了 kubeadm init 以及 kubeadm join 这两个命令作为快速创建 kubernetes 集群的最佳实践。
kubeadm 通过执行必要的操作来启动和运行一个最小可用的集群。它被故意设计为只关心启动集群，而不是之前的节点准备工作。同样的，诸如安装各种各样值得拥有的插件，例如 Kubernetes Dashboard、监控解决方案以及特定云提供商的插件，这些都不在它负责的范围。
相反，我们期望由一个基于 kubeadm 从更高层设计的更加合适的工具来做这些事情；并且，理想情况下，使用 kubeadm 作为所有部署的基础将会使得创建一个符合期望的集群变得容易。

以下是简述使用kubeadm搭建一个集群的过程

<!--more-->

## 系统准备
### 确认系统信息
为了保证实验顺利进行，使用了以下系统，如使用其他系统，请自行查询[官方文档](https://kubernetes.io/docs/home/)
系统信息：
```shell
[user@S001 ~]# cat /etc/redhat-release 
CentOS Linux release 7.6.1810 (Core)
[user@S001 ~]# uname -r
3.10.0-957.21.3.el7.x86_64
```
### 修改主机名称
有些主机名称会造成创建集群时报错，所以要进行主机名称修改
```shell
[user@S001 ~]# hostnamectl --set-hostname --static master001
```

### 修改/etc/hosts
目的的在局域网中能通过主机名称解释到IP地址
```shell
[root@CATS001 ~]# vim /etc/hosts
::1     localhost       localhost.localdomain   localhost6      localhost6.localdomain6
127.0.0.1       localhost       localhost.localdomain   localhost4      localhost4.localdomain4
192.168.1.32    CATS001 CATS001
192.168.1.32    k8s.zb.vip
```
如有其他节点也可以添加在里面，这里我只有一个节点

### 关闭防火墙
为了避免防火墙规则造成的错误，对其进行关闭
```shell
[user@CATS001 ~]# systemctl stop firewalld && systemctl disable firewalld
```

### 禁用selinux
临时禁用，系统重启会失效
```shell
setenforce 0
```
永久关闭，通过修改/etc/selinux/config文件
```shell
sed -i s/SELINUX=enforcing/SELINUX=disabled/g /etc/selinux/config 
```

### 关闭SWAP	
kubernetes 1.8开始要求关闭swap，如果启用了会出现报错
```shell
swapoff -a
vm.swappiness=0
```

### 开启IPVS
```shell
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
ipvs_modules="ip_vs ip_vs_lc ip_vs_wlc ip_vs_rr ip_vs_wrr ip_vs_lblc ip_vs_lblcr ip_vs_dh ip_vs_sh ip_vs_fo ip_vs_nq ip_vs_sed ip_vs_ftp nf_conntrack"
for kernel_module in \${ipvs_modules}; do
/sbin/modinfo -F filename \${kernel_module} > /dev/null 2>&1
if [ $? -eq 0 ]; then
/sbin/modprobe \${kernel_module}
fi
done
EOF
```
检查IPVS开启信息
```shell
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep ip_vs
```

### 修改内核参数
配置L2网桥在转发包时会被iptables的FORWARD规则所过滤
```shell
echo """
vm.swappiness = 0
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
""" >> /etc/sysctl.conf
```
使设置的内核参数生效，centos7添加bridge-nf-call-ip6tables出现[No such file or directory](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.cnblogs.com%2Fzejin2008%2Fp%2F7102485.html),简单来说就是执行一下 modprobe br_netfilter
```shell
sysctl -p
```

修复ipvs模式下长连接timeout问题 小于900即可
```shell
cat <<EOF > /etc/sysctl.d/k8s.conf
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 10
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
net.ipv4.neigh.default.gc_stale_time = 120
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.default.arp_announce = 2
net.ipv4.conf.lo.arp_announce = 2
net.ipv4.conf.all.arp_announce = 2
net.ipv4.ip_forward = 1
net.ipv4.tcp_max_tw_buckets = 5000
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 1024
net.ipv4.tcp_synack_retries = 2
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.netfilter.nf_conntrack_max = 2310720
fs.inotify.max_user_watches=89100
fs.may_detach_mounts = 1
fs.file-max = 52706963
fs.nr_open = 52706963
net.bridge.bridge-nf-call-arptables = 1
vm.swappiness = 0
vm.overcommit_memory=1
vm.panic_on_oom=0
EOF
```
使内核参数生效
```shell
sysctl --system
```

### 安装Docker 
安装过程打印的信息就不提供了
```shell
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum makecache fast
yum list docker-ce --showduplicates | sort -r
yum install docker-ce-<VERSION_STRING>
```
* 安装Docker依赖包
* 增加Docker yum源
* 将服务器上的软件包信息 现在本地缓存,以提高 搜索 安装软件的速度
* 列出Docker版本
* 安装指定Docker指定版本，安装最新版本直接运行yum -y install -y install docker-ce
修改Docker启动参数
```shell
sed -i "13i ExecStartPost=/usr/sbin/iptables -P FORWARD ACCEPT" /usr/lib/systemd/system/docker.service
mkdir -p /etc/docker
cat > /etc/docker/daemon.json <<EOF
{
"exec-opts": ["native.cgroupdriver=systemd"],
"log-driver": "json-file",
"log-opts": {
"max-size": "100m"
},
"storage-driver": "overlay2",
"registry-mirrors": ["https://uyah70su.mirror.aliyuncs.com"]
}
EOF
```
设置Docker开机启动
```shell
systemctl daemon-reload
systemctl enable docker
systemctl start docker
```
### 安装Kubernetes相关组件
增加Kubernetes yum源
```shell
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```
安装kuberlet kuberadm kubectl
```shell
yum install -y kubelet kubeadm kubectl
```
设置kubelet开机启动
```shell
systemctl enable kubelet
```

## 通过kubradm初始化集群
### 编写节点配置文件
```shell
echo """
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.15.4
imageRepository: registry.aliyuncs.com/google_containers
controlPlaneEndpoint: "k8s.zb.vip:6443"
apiServer:
  certSANS:
  - "k8s.zb.vip"
networking:
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.88.0.0/16
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
""" > /etc/kubernetes/kubeadm-config.yaml
```
* 指定Kubernetes版本
* 指定apiServer，这里使用了域名，主要是为以后高可用做准备
* 指定kube-proxy为IPVS模式，默认为iptbles

### 初始化master的kubelet
```shell
kubeadm init --config=/etc/kubernetes/kubeadm-config.yaml --upload-certs | tee kubeadm-init.log

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join k8s.xiaobu.vip:6443 --token 8lp09d.m961tfj5cctkq76s \
    --discovery-token-ca-cert-hash sha256:f0553763add9e232b4eebde18b018f5cd09a0c43afe6f490e38d0f30ba4f2708 \
    --control-plane --certificate-key 305bb07984ed0f9a552fa0a4b172163f5cc1f772f13ad4b8daf983bc35eb0547

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use 
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join k8s.xiaobu.vip:6443 --token 8lp09d.m961tfj5cctkq76s \
    --discovery-token-ca-cert-hash sha256:f0553763add9e232b4eebde18b018f5cd09a0c43afe6f490e38d0f30ba4f2708
```
出现以上字段表示集群安装成功

### 配置kubectl操作集群
```shell
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```
### 设置kubectl自动补全命令
安装bash-completion
```shell
yum install bash-completion -y
```
设置kubectl自动补全开机自动
```shell
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
source ~/.bash_profile
```
### 安装集群网络插件
这里使用kube-flanel插件，使用其他插件请自行谷歌或者百度
```shell
kubectl apply -f  https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
### 验证集群
```shell
[root@CATS001 kubernetes]# kubectl get pods --all-namespaces
NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
kube-system   coredns-bccdc95cf-gktjn              1/1     Running   0          5m24s
kube-system   coredns-bccdc95cf-wntxb              1/1     Running   0          5m24s
kube-system   etcd-cats001                         1/1     Running   0          4m44s
kube-system   kube-apiserver-cats001               1/1     Running   0          4m19s
kube-system   kube-controller-manager-cats001      1/1     Running   1          4m33s
kube-system   kube-flannel-ds-amd64-9jsq9          1/1     Running   0          2m24s
kube-system   kube-proxy-b6t5d                     1/1     Running   0          5m25s
kube-system   kube-scheduler-cats001               1/1     Running   1          4m21s
[root@CATS001 kubernetes]# kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-0               Healthy   {"health":"true"}
[root@CATS001 kubernetes]# kubectl get nodes
NAME         STATUS   ROLES    AGE    VERSION
cats001      Ready    master   8m4s   v1.15.2
```
所有Pod STATUS都为Running

## 如是单master节点请忽略以下操作
### 修改/etc/hosts文件
```shell
[root@CATS001 kubernetes]# cat /etc/hosts
::1	localhost	localhost.localdomain	localhost6	localhost6.localdomain6
127.0.0.1	localhost	localhost.localdomain	localhost4	localhost4.localdomain4
192.168.1.27    S001
192.168.1.25    S002
192.168.1.26    S003

192.168.1.25	k8s.zb.vip
192.168.1.25    k8s.zb.vip
192.168.1.26    k8s.zb.vip
```


### 另外两台主机部署为master节点
两台主机都运行
```shell
kubeadm join k8s.zb.vip:6443 --token 8lp09d.m961tfj5cctkq76s --discovery-token-ca-cert-hash sha256:f0553763add9e232b4eebde18b018f5cd09a0c43afe6f490e38d0f30ba4f2708 --control-plane --certificate-key 305bb07984ed0f9a552fa0a4b172163f5cc1f772f13ad4b8daf983bc35eb0547
```
此命令和Node节点加入集群在第一个master节点安装成功时打印出来留意并做好保存

## 其他问题
### master node去除污点限制
```shll
kubectl describe node master01 | grep Taint
kubectl taint nodes master01 node-role.kubernetes.io/master- #去除污点
kubectl taint nodes master01 node-role.kubernetes.io/master=:NoSchedule    #设置污点
```

### 重置master，node节点
集群初始化如果遇到问题，可以使用下面的命令进行清理：
```shell
kubeadm reset -f
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
ipvsadm --clear
systemctl stop kubelet
systemctl stop docker
rm -rf /var/lib/cni/
rm -rf /var/lib/kubelet/*
rm -rf /etc/cni/
ifconfig cni0 down
ifconfig flannel.1 down
ifconfig docker0 down
ip link delete cni0
ip link delete flannel.1
systemctl start docker
```

###  pod无法访问api-service
```shell
iptables -t nat -I POSTROUTING -s 10.244.0.0/16 -j MASQUERADE
```
其中10.244.0.0/16为POD使用的网络

### 生产环境node移除
```shell
kubectl cordon node01 #标记node01不可调度
kubectl drain node2 --delete-local-data --force --ignore-daemonsets
kubectl uncordon node01 #node01可重新调度
在node上执行
kubeadm reset
ifconfig cni0 down
ip link delete cni0
ifconfig flannel.1 down
ip link delete flannel.1ip
rm -rf /var/lib/cni/
在master上执行
kubectl delete node node01
```

### node忘记kubeadm join的解决方法
```shell
重新生成一条永久的token
在master节点上运行
kubeadm token create --ttl 0
kubeadm token list
获取ca证书sha256编码hash值
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
kubeadm join masterIP:6443 --token xxxxxx --discovery-token-ca-cert-hash xxxxx
主节点忘记
kubeadm init phase upload-certs --upload-certs
```

### DNS测试
```shell
kubectl run curl --rm --image=radial/busyboxplus:curl -it
nslookup kubernetes.default
```

### Pod启动报错排查
主要有两个命令
```shell
kubectl describe pods <PodName> -n <Namespace>
kubectl logs <PodName> -n <Namespace>
```
如果Pod中有多个容器使用
```shell
kubectl logs <PodName> -n <Namespace> -c <ContainerName>
```

## 结语
kubeadm部署Kubernetes集群就介绍到这么多，如有疑问可以添加我的微信，或者自行谷歌百度进行解决。
