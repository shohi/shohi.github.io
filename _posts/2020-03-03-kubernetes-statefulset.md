---
layout: post
title: Kubernetes之StatefulSet
category: tech
guid: 67940C3D-996F-41FE-96AF-838AA77ED4F9
tags: [tech, k8s, cloud, golang]

---
{% include JB/setup %}

`StatefulSet`是Kubernetes(K8s)中的一种资源类型, StatefulSets 旨在与有状态的应用及分布式系统一起使用[2]。以StatefulSet方式部署的
应用将有以下特点,

```
1. 固定的唯一网络标识，即Hostname主机名
2. 按顺序创建并按反序删除
3. 固定的持久化存储, 即每个Pod有自己对应的PVC及PV
4. 按反序滚动更新
```

这些是其基本特性, 在后续的版本中可能对其进行扩展[2].

## 1. 创建K8s Cluster测试环境

常见的创建本地K8s集群方式有--`minikube`/`kind`/`k3s`,Max Brenner在[这篇](https://brennerm.github.io/posts/minikube-vs-kind-vs-k3s.html)文章中详细介绍了它们的区别及使用场景.

个人认为最为便捷的方式是使用`kind`, 创建十分简单.

```
# install `kind` cli
brew install kind

# create k8s cluster with specified Kubernetes version
kind create cluster --name local --image kindest/node:v1.12.9
```
上述命令将创建名为`kind-local`的K8s cluster, 且版本为v1.12.9. `kindest/node:v1.12.9`是封装了K8s的docker image.

## 2. kubectx

`kubectx`用来快速地切换`kubectl`所作用的默认context(cluster)及其namespace.

```
brew install kubectx

# show the current context
kubect --current

# switch to `kind-local` context
kubectx kind-local
```

### 3. StatefulSet创建

创建前, 首先查看当前K8s支持的资源类型及其版本.

```
# list supported resources
#   - kubectl api-resources -h
kubectl api-resources

# list supported API versions on the server
kubectl api-versions
```

`StatefulSet`的基本帮助信息可通过下面命令查看.

```
# or use abbrev form - `sts`, e.g. `kubectl explain sts`
kubectl explain statefulset --api-version apps/v1
```

确定过版本之后, 创建如下YAML样例文件--`sts-sample.yaml`(参考`Kubernetes in Action`第10章), 注意`kind`为`StatefulSet`.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: ale
spec:
  serviceName: ale
  replicas: 2
  selector:
    matchLabels:
      app: ale
  template:
    metadata:
      labels:
        app: ale
    spec:
      containers:
      - name: main
        image: alpine:3.11
        command: ["/bin/sh"]
        args: ["-c", "tail -f /dev/null"]
        ports:
        - name: http
          containerPort: 8080
```

使用如下命令在当前K8s上创建一个StatefulSet

```
kubectl create -f sample-sts.yaml
```


### 4. StatefulSet管理

查看StatefulSet及其关联的Pod

```
# list all statefulsets
kubectl get sts
|> NAME   DESIRED   CURRENT   AGE
|> ale    2         2         23m

# list pods filtered by label
kubectl get pod -l app=ale
|> NAME    READY   STATUS    RESTARTS   AGE
|> ale-0   1/1     Running   0          24m
|> ale-1   1/1     Running   0          24m

# describe given statefulset
kubectl describe sts ale

# describe pod
kubectl describe pod ale-0
```

更新StatefulSet

```
# use `edit`
kubectl edit sts ale

# use `patch` with josn format
kubectl patch sts ale -p '{"spec":{"replicas":3}}'

# view updates
kubect get pods -l app=ale
NAME    READY   STATUS              RESTARTS   AGE
|> ale-0   1/1     Running             0          28m
|> ale-1   1/1     Running             0          28m
|> ale-2   0/1     ContainerCreating   0          2s
```

删除StatefulSet

```
# use file
kubectl delete -f [STS_YAML_FILE]

# or delete resource directly
kubectl delete sts [STS_NAME]
```

## 5. Tips

- 向Pod中的Container传递信息

常见的有三种 -- 环境变量, ConfigMap/Secret及Downward API. Downward API可以将Pod的metadata传递给其内部的container, 比如Node名称, Pod名称/IP/Label等.

```yaml
# omit...
spec:
  containers:
  - name: main
    image: alpine:3.11
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    # omit...
```

这些可以传递的变量名称是可以通过`kubect get pod ale-0 -o json`来得到一个大致的印象, 但是引用方式不一定完全按照json中的field名称(如cpu.requests/cpu.limits), 详细信息参考[Downward API文档](https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/).

需要注意的是环境变量是Container级别, 同一个POD内Container[不共享环境变量](https://stackoverflow.com/questions/50203731/is-it-possible-for-2-containers-inside-a-pod-to-share-the-same-environment-varia). 更多关于ConfigMap及Downward API信息，可参考`Kubernetes in Action`第7/8章.

- StatefulSet自动添加的label

StatefulSet在创建新的Pod时, 会对[pod自动添加一个label](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#pod-name-label), 这个可用于`selector`.

```
statefulset.kubernetes.io/pod-name=[$(statefulset name)-$(ordinal)]
```

## Note
1. K8s版本-`v1.12.9`
2. Kubernetes StatefulSet tutorial, <https://kubernetes.io/zh/docs/tutorials/stateful-application/basic-stateful-set/>
3. Kuberentes StatefulSet document, <https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/>
4. Kubernetes in Action, <https://www.manning.com/books/kubernetes-in-action>
5. Kubernetes in Docker(kind), <https://github.com/kubernetes-sigs/kind>
6. kubectx, <https://github.com/ahmetb/kubectx>
7. kubectl plugins, <https://github.com/kubernetes-sigs/krew-index/blob/master/plugins.md>
