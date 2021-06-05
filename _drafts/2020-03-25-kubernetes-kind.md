---
layout: post
title: Kubernetes之Kind
category: tech
guid: 3B288ACF-F312-4BC0-8BD6-2C4715F31154
tags: [tech, k8s]

---
{% include JB/setup %}

在介绍K8s Statefulset中曾提及`kind`, `kind`是一种用来本地搭建K8s环境的工具, 其仅依赖于docker, 用起来十分方便. 本文将进一步介绍它的用法, 更详细的使用请阅读官方文档[kind](https://github.com/kubernetes-sigs/kind).

## cluster创建及删除

* 创建cluster
```
kind create cluster --name test --image kindest/node:v1.12.9
```

* 删除cluster

```
kind delete cluster --name test
```




## image加载

加载本地docker image到k8s cluster - `kind load`.

```
kind load docker-image -v 3 --name [cluster-name] redis:5.0.7
```

上述命令将加载`redis:5.0.7`这个docker image到k8s cluster, 这里加载是指让kind所创建的k8s能访问到, `-v`用于输出详细的log. (不清楚其会不会拷贝image文件, 感觉创建链接更为合理, 需要查看文档或者阅读源码)
