---
title: Kubernetes之使用Kustomize配置应用
subtitle: Kubernetes应用篇 
date: 2019-10-25
tags: ["kubernetes", "helm", "kustomize"]
keywords: ["kubernetes", "helm", "kustomize", "模板"]
slug: kubernetes-kustomize
gitcomment: false
bigimg: [{src: "https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/photo-1565780421727-543646cefafb.jpeg", desc: ""}]
category: "kubernetes"
---

如果你经常使用 Kubernetes，那么你肯定就有定制资源清单文件的需求，但是貌似现在大家都比较喜欢使用 Helm，Helm 很好用，但也有很多缺点，比如需要一个 tiller 服务端，需要超高的权限，最重要的是如果你要想自己做一个 Helm Chart 包的话，则不是那么容易的，需要你了解一些 go template 的相关知识，它抛弃了我们在 Docker 和 Kubernetes 上面学到的一些逻辑，今天我们将为大家介绍另外一种名为`Kustomize`的替代工具。

<!--more-->

实际上 Kustomize 并不是一个新的工具，而且现在已经被集成在了 kubectl 1.14 版本的子命令中了，是不是非常方便了，免去了安装第三方工具的麻烦，因为 kubectl 工具基本上是我们天天都在使用的，所以......你可以把 Helm 命令扔掉了。

Kustomize 和 Kubernetes 一样，它完全就是声明式的，你说你想要什么，系统就提供给你什么，不需要遵循命令方式来描述你希望构建的对象。

其次，它和 Docker 比较类似，有很多层组成，每个层都是修改以前的层，正因为有这个理念存在，所以我们可以不断在其他人至上写东西，而不会增加配置的复杂性，构建的最终结果由基础部分和你在上面配置的其他层组成。

最后，和 Git 一样，你可以使用一个远程的基础配置作为最原始的配置，然后在该基础上添加一些自定义的配置。

## 安装
对于Linux 用户来说，那么你就可以直接从 [Release 页面](https://github.com/kubernetes-sigs/kustomize/releases)上面下载二进制文件然后添到 PATH 路径下面即可：
```shell
curl -s https://api.github.com/repos/kubernetes-sigs/kustomize/releases | grep browser_download | grep linux | cut -d '"' -f 4 |grep /kustomize/v | sort | tail -n 1 | xargs curl -O -L
tar xzf ./kustomize_v*_linux_amd64.tar.gz
./kustomize version
```

当然如果你愿意也可以从源码中直接构建，代码仓库：[https://github.com/Kubernetes-sigs/kustomize](https://github.com/Kubernetes-sigs/kustomize)。

## 基础模板
要使用 Kustomize，你需要有一个原始的 yaml 文件来描述你想要部署到集群中的任何资源，我们这里将这些 base 文件存储在`./k8s/base/`文件夹下面。

这些文件我们**永远**不会直接访问，我们将在它们上面添加一些自定义的配置来创建新的资源定义。

> 你可以在任何时间点使用`kubectl apply -f ./k8s/base/`命令来构建基础模板。

下面例子中，我们将使用 Service 和 Deployment 资源对象为例进行说明。下面定义两个资源清单文件：
service.yaml 定义如下:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: sl-demo-app
spec:
  ports:
    - name: http
      port: 8080
  selector:
    app: sl-demo-app
```

deployment.yaml 定义如下：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sl-demo-app
spec:
  selector:
    matchLabels:
      app: sl-demo-app
  template:
    metadata:
      labels:
        app: sl-demo-app
    spec:
      containers:
      - name: app
        image: foo/bar:latest
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
```


然后在当前文件夹下面添加一个名为`kustomization.yaml`的文件：
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - service.yaml
  - deployment.yaml
```

这个文件将是你的基础配置文件，它描述了你使用的资源文件。

> 当你运行`kubectl apply -f ./k8s/base/`命令时，该`kustomization.yaml`文件可能会出现一些错误，你可以添加参数`--validate=false`进行校验，当然也可以不针对整个文件夹运行该命令。

要将基础模板中的资源安装到你的集群中，只需要执行以下命令即可：
```shell
$ kubectl apply -k k8s/base
service/sl-demo-app created
deployment.apps/sl-demo-app created
```

为了了解将安装什么资源到集群中，我们在本文中主要使用`kustomize build`命令来代替`kubectl apply -k`命令。当然使用`kubectl kustomize`命令也是可以的，因为我们说了 kubectl 1.14 版本以后就已经集成了 kustomize。
<!--adsense-text-->
使用`kustomize build`命令运行后的结果如下所示，我们会看到两个文件连接在一起：
```shell
$ kustomize build k8s/base
apiVersion: v1
kind: Service
metadata:
  name: sl-demo-app
spec:
  ports:
  - name: http
    port: 8080
  selector:
    app: sl-demo-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sl-demo-app
spec:
  selector:
    matchLabels:
      app: sl-demo-app
  template:
    metadata:
      labels:
        app: sl-demo-app
    spec:
      containers:
      - image: foo/bar:latest
        name: app
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
```


## 定制
现在我们想要针对一些特定场景进行定制，比如，针对生产环境和测试环境需要由不同的配置。我们这里并不会涵盖 Kustomize 的整个功能集，而是作为一个标准示例，向你展示这个工具背后的哲学。

首先我们创建一个新的文件夹`k8s/overlays/prod`，其中包含一个名为`kustomzization.yaml`的文件，文件内容如下：
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
- ../../base
```

当前文件夹下面的目录结构如下所示：
```shell
$ tree
.
└── k8s
    ├── base
    │   ├── deployment.yaml
    │   ├── kustomization.yaml
    │   └── service.yaml
    └── overlays
        └── prod
            └── kustomization.yaml
```

如果现在我们构建这个文件，将会看到和之前构建 base 目录一样的结果：
```shell
$ kustomzie build k8s/overlays/prod
apiVersion: v1
kind: Service
metadata:
  name: sl-demo-app
spec:
  ports:
  - name: http
    port: 8080
  selector:
    app: sl-demo-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sl-demo-app
spec:
  selector:
    matchLabels:
      app: sl-demo-app
  template:
    metadata:
      labels:
        app: sl-demo-app
    spec:
      containers:
      - image: foo/bar:latest
        name: app
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
```

接下来我们来为我们的`prod`环境进行一些定制。

### 定义环境变量
在 base 基础模板中，我们定义任何环境变量，现在我们需要添加一些环境变量在之前的基础模板中。实际上很简单，我们只需要在我们的基础模板上创建一块我们想要模板化的代码块，然后将其引用到`kustomization.yaml`文件中即可。

比如我们这里定义一个包含环境变量的配置文件：(custom-env.yaml)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sl-demo-app
spec:
  template:
    spec:
      containers:
        - name: app # (1)
          env:
            - name: CUSTOM_ENV_VARIABLE
              value: Value defined by Kustomize
```

> 注意 (1) 这里定义的 name 是非常重要的，kustomize 会通过该值找到需要修改的容器。

这个 yaml 文件本身是无效的，它只描述了我们希望在上面的基础模板上添加的内容。我们只需要将这个文件添加到`k8s/overlays/prod/kustomization.yaml`文件中即可：
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
- ../../base

patchesStrategicMerge:
- custom-env.yaml

```

现在如果我们来构建下，可以看到如下的输出结果：
```shell
$ kustomize build k8s/overlays/prod
apiVersion: v1
kind: Service
metadata:
  name: sl-demo-app
spec:
  ports:
  - name: http
    port: 8080
  selector:
    app: sl-demo-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sl-demo-app
spec:
  selector:
    matchLabels:
      app: sl-demo-app
  template:
    metadata:
      labels:
        app: sl-demo-app
    spec:
      containers:
      - env:
        - name: CUSTOM_ENV_VARIABLE
          value: Value defined by Kustomize
        image: foo/bar:latest
        name: app
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
```

可以看到我们的 env 块已经被合并到了我们的基础模板上了，自定义的 env 变量出现在了 deployment.yaml 文件中。


### 修改副本数量
和上面的例子一样，我们来扩展我们的基础模板来定义一些还没有定义的变量。

> 你也可以覆盖一些在 base 文件中已有的变量。

这里我们来添加一些关于副本的信息，和前面一样，只需要在一个 YAML 文件中定义副本所需的额外信息块，新建一个名为`replica-and-rollout-strategy.yaml` 的文件，内容如下所示：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sl-demo-app
spec:
  replicas: 10
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
```

和前面一样，在`kustomization.yaml`文件中的`patchesStrategicMerge`下面添加这里定制的数据：
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
- ../../base

patchesStrategicMerge:
- custom-env.yaml
- replica-and-rollout-strategy.yaml
```

同样，这个时候再使用`kustomize build`命令构建，如下所示：
```shell
$ kustomize build k8s/overlays/prod
apiVersion: v1
kind: Service
metadata:
  name: sl-demo-app
spec:
  ports:
  - name: http
    port: 8080
  selector:
    app: sl-demo-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sl-demo-app
spec:
  replicas: 10
  selector:
    matchLabels:
      app: sl-demo-app
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: sl-demo-app
    spec:
      containers:
      - env:
        - name: CUSTOM_ENV_VARIABLE
          value: Value defined by Kustomize
        image: foo/bar:latest
        name: app
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
```

我们可以看到副本数量和滚动更新的策略都添加到了基础模板之上了。

### 通过命令行定义 secret
我们常常会通过命令行来添加一个 secret 对象，`kustomize`有一个`edit`的子命令可以用来编辑`kustomization.yaml`文件然后创建一个 secret 对象，比如我们这里添加一个如下所示的 secret 对象：

```shell
$ cd k8s/overlays/prod
$ kustomize edit add secret sl-demo-app --from-literal=db-password=12345
```

上面的命令会修改`kustomization.yaml`文件添加一个`SecretGenerator`字段在里面。

> 当然你也可以通过文件（比如`--from-file=file/path`或者`--from-evn-file=env/path.env`）来创建 secret 对象。

通过上面命令创建完 secret 对象后，`kustomization.yaml`文件的内容如下所示：
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
- ../../base

patchesStrategicMerge:
- custom-env.yaml
- replica-and-rollout-strategy.yaml
secretGenerator:
- literals:
  - db-password=12345
  name: sl-demo-app
  type: Opaque
```

然后同样的我们回到根目录下面执行`kustomize build`命令构建下模板，输出内容如下所示：
```shell
$ kustomize build k8s/overlays/prod
apiVersion: v1
data:
  db-password: MTIzNDU=
kind: Secret
metadata:
  name: sl-demo-app-6ft88t2625
type: Opaque
---
apiVersion: v1
kind: Service
metadata:
  name: sl-demo-app
spec:
  ports:
  - name: http
    port: 8080
  selector:
    app: sl-demo-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sl-demo-app
spec:
  replicas: 10
  selector:
    matchLabels:
      app: sl-demo-app
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: sl-demo-app
    spec:
      containers:
      - env:
        - name: CUSTOM_ENV_VARIABLE
          value: Value defined by Kustomize
        image: foo/bar:latest
        name: app
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
```

> 我们可以看到 secret 对象的名称是`sl-demo-app-6ft88t2625`，而不是我们定义的`sl-demo-app`，这是正常的，因为如果更改了 secret 内容，就可以触发滚动更新了。

同样的，如果我们想要在 Deployment 中使用这个 Secret 对象，我们就可以像之前一样添加一个使用 Secret 的新的层定义即可。

比如我们这里像把`db-password`的值通过环境变量注入到 Deployment 中，我们就可以定义下面这样的新的层信息：（database-secret.yaml）
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sl-demo-app
spec:
  template:
    spec:
      containers:
      - name: app
        env:
        - name: "DB_PASSWORD"
          valueFrom:
            secretKeyRef:
              name: sl-demo-app
              key: db.password
```

然后同样的，我们把这里定义的层添加到`k8s/overlays/prod/kustomization.yaml`文件中去：
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
- ../../base

patchesStrategicMerge:
- custom-env.yaml
- replica-and-rollout-strategy.yaml
- database-secret.yaml

secretGenerator:
- literals:
  - db-password=12345
  name: sl-demo-app
  type: Opaque
```

现在我们来构建整个的 prod 目录，我们会得到如下所示的信息：
```shell
$ kustomize build k8s/overlays/prod
apiVersion: v1
data:
  db-password: MTIzNDU=
kind: Secret
metadata:
  name: sl-demo-app-6ft88t2625
type: Opaque
---
apiVersion: v1
kind: Service
metadata:
  name: sl-demo-app
spec:
  ports:
  - name: http
    port: 8080
  selector:
    app: sl-demo-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sl-demo-app
spec:
  replicas: 10
  selector:
    matchLabels:
      app: sl-demo-app
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: sl-demo-app
    spec:
      containers:
      - env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              key: db.password
              name: sl-demo-app-6ft88t2625
        - name: CUSTOM_ENV_VARIABLE
          value: Value defined by Kustomize
        image: foo/bar:latest
        name: app
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
```

我们可以看到`secretKeyRef.name`的值也指定的被修改成了上面生成的 secret 对象的名称。

> 由于 Secret 是一些私密的信息，所以最好是在安全的环境中来添加上面的 secret 的对象，而不应该和其他代码之类的一起被提交到代码仓库之类的去。

如果是 ConfigMap 的话也是同样的逻辑，最后会生成一个 hash 值的名称，这样在 ConfigMap 更改时可以触发重新部署。


### 修改镜像
和 secret 资源对象一样，我们可以直接从命令行直接更改镜像或者 tag，如果你需要部署通过 CI/CD 系统标记的镜像的话这就非常有用了。

比如我们这里来修改下镜像的 tag：
```shell
$ cd k8s/overlays/prod
$ TAG_VERSION=3.4.5
$ kustomize edit set image foo/bar=foo/bar:$TAG_VERSION
```

> 一般情况下`TAG_VERSION`常常被定义在 CI/CD 系统中。

现在的`kustomization.yaml`文件内容如下所示：
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
- ../../base

patchesStrategicMerge:
- custom-env.yaml
- replica-and-rollout-strategy.yaml
- database-secret.yaml

secretGenerator:
- literals:
  - db-password=12345
  name: sl-demo-app
  type: Opaque

images:
- name: foo/bar
  newName: foo/bar
  newTag: 3.4.5
```

同样回到根目录下面构建该模板，会得到如下所示的信息：
```shell
$ kustomize build k8s/overlays/prod
apiVersion: v1
data:
  db-password: MTIzNDU=
kind: Secret
metadata:
  name: sl-demo-app-6ft88t2625
type: Opaque
---
apiVersion: v1
kind: Service
metadata:
  name: sl-demo-app
spec:
  ports:
  - name: http
    port: 8080
  selector:
    app: sl-demo-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sl-demo-app
spec:
  replicas: 10
  selector:
    matchLabels:
      app: sl-demo-app
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: sl-demo-app
    spec:
      containers:
      - env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              key: db.password
              name: sl-demo-app-6ft88t2625
        - name: CUSTOM_ENV_VARIABLE
          value: Value defined by Kustomize
        image: foo/bar:3.4.5
        name: app
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
```

我们可以看到 Deployment 的第一个`container.image`已经被修改了 3.4.5 版本了。

最终我们定制的模板文件目录结构如下所示：
```shell
$ tree .
.
└── k8s
    ├── base
    │   ├── deployment.yaml
    │   ├── kustomization.yaml
    │   └── service.yaml
    └── overlays
        └── prod
            ├── custom-env.yaml
            ├── database-secret.yaml
            ├── kustomization.yaml
            └── replica-and-rollout-strategy.yaml

4 directories, 7 files
```

要安装到集群中也很简单：
```shell
$ kustomize build k8s/overlays/prod | kubectl apply -f -
```

## 总结
在上面的示例中，我们了解到了如何使用 Kustomize 的强大功能来定义你的 Kuberentes 资源清单文件，而不需要使用什么额外的模板系统，创建的所有的修改的块文件都将被应用到原始基础模板文件之上，而不用使用什么花括号之类的修改来更改它。

Kustomize 中还有很多其他高级用法，比如 mixins 和继承或者允许为每一个创建的对象定义一个名称、标签或者 namespace 等等，你可以在官方的 [Kustomize GitHub 代码仓库](https://github.com/kubernetes-sigs/kustomize)中查看高级示例和文档。

<!--adsense-self-->

## 参考
* https://www.qikqiak.com/post/kustomize-101/
