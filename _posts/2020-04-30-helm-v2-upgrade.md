---
layout: post
title: helm v2之upgrade
category: tech
guid: 368CAFF4-6B93-4428-A447-22A3BDD07C78
tags: [tech, k8s, helm]

---
{% include JB/setup %}

`helm`中用于升级部署的应用的子命令是`upgrade`, 该命令可以说是日常运营中最经常使用的. 但是升级同时也暗含着一个问题-即如何降级. 当升级出错时, helm应该能将应用回滚到升级前的状态并保留升级过程中的日志信息, 以便排查. 本文将简要介绍`helm upgrade`以及其对应的回滚子命令`helm rollback`的一些常用的用法.

## 1. 查看Release的历史版本

```
$> helm history [release-name]

REVISION	UPDATED                 	STATUS    	...
1       	Fri Mar  6 11:13:19 2020	SUPERSEDED
2       	Fri Mar  6 11:18:47 2020	DEPLOYED
```

## 2. 升级release

升级意味着配置文件有更新，需要将这些更新应用到release上. 更新配置文件通常有两种方式 - `命令行`和`配置文件`.

### 2.1 命令行修改配置

```
$> helm upgrade [release-name] [chart-name] --set foo=bar --set foo=newbar
```

### 2.2 配置文件

```
$> helm upgrade [realse-name] [chart-name] -f [values.yaml] -f [values.patch.yaml]
```

## 3. 回退release

升级后如果发现新升级的系统有问题, 那么就需要回退到之前可用的某次部署上.`rollback`子命令就是用来将release回滚到之前的版本.

```
helm rollback <RELEASE> [REVISION] [flags]
```
其中`REVISION`可由`helm history`得到.

另外有时想对比不同REVISION之间的差异，可用[helm-diff](https://github.com/databus23/helm-diff)这个插件来.

```
# install plugin
# helm plugin install https://github.com/databus23/helm-diff --version master
helm plugin install https://github.com/databus23/helm-diff --version v2.9.0+3

# release diff
helm diff revision [RELEASE] [REV_1] [REV_2]
```
注意`helm-diff`的版本需要与`helm`兼容, 由于测试用的是`helm v2.9.1`, 所以选了`v2.9.0+3`.
