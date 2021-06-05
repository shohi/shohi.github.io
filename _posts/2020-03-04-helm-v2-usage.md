---
layout: post
title: Helm v2使用
category: tech
guid: DF3FFFD4-4E7A-477A-AAA5-7FA154C78F44
tags: [tech, k8s, helm, cloud, golang]

---
{% include JB/setup %}

`Helm`是K8s应用部署及管理工具, 相对于`kubectl`作用于单一的资源, `Helm`的操作对象是应用.
一般来说应用不仅仅包括`deployment`, 还有相应的配置、权限管理以及与其关联的`service`及`endpoint`.

`helm`中将应用抽象成一个`chart`, `chart`就是应用所需的各种资源文件集合. `Helm`将其中不变化的部分
抽象成模板, 易变的部分提取到一个yaml文件中--`values.yaml`, `Helm`利用这两种文件生成K8s所需要的各种资源以及管理应用.

每次更新`values.yaml`时, `Helm`就生成一个新版本的部署文件，同时监控整个更新过程.
如果有错误发生，`Helm`就回滚到先前的状态. 对于用户来说，只需要维护一个描述文件--`Values.yaml`就可以了,
很大程度上减轻了K8s应用部署及管理的难度.

## Helm安装
推荐使用`Helm`版本管理工具-[helmenv](https://github.com/little-angry-clouds/helmenv)来安装, 因为其能很方便地切换不同版本.

```bash
# clone `helmenv` into `~/.helm`
mkdir -p ~/.helm
git clone https://github.com/little-angry-clouds/helmenv.git ~/.helm

# source the script
source $HOME/.helm/helmenv.sh

# add ~/.helm to $PATH
export PATH="$HOME/.helm:$PATH"
```

使用`helmenv`安装`Helm`.

```bash
# list all available versions
helmenv list-remote

# install specific version
helmenv install v2.16.3

# use specific version
helmenv use v2.16.3
```

安装完helm之后, 最好加载[脚本自动补全(autocompletion)](https://helm.sh/docs/helm/helm_completion/#scrollpane), 按下`TAB`键会显示相应的提示. ([这篇文章](https://iridakos.com/programming/2018/03/01/bash-programmable-completion-tutorial)详细地解释了自动补全的机制)

```bash
# source <(helm completion bash)
source <(helm completion zsh)
```

## Helm初始化
Helm在v2版本需要先在K8s部署其Server--Tiller, 然后才能使用`helm`命令行来管理应用. 在v3版本中, 移除了Tiller依赖, 即Helm不需要先与定制的Server通信，然后由Server来与K8s API通信对应用进行管理. 也就说v3中, `helm`可以直接拿来使用，不要任何初始化.

`v3`相较于`v2`有不少的改进, 对于新应用应尽可能使用v3. 鉴于在工作中, 仍有不少的应用用v2版本管理, 本文仍基于v2.  关于v3的特性以及如何迁移, 官网给出了详细说明--[官网blog](https://helm.sh/blog/helm-3-released/)和[FAQ](https://helm.sh/docs/faq/).

安装`Tiller`

```bash
# only for helm v2
helm init
```

安装完tiller之后, 还需要对tiller关联的`ServiceAccount`进行设置, 否则[tiller无权操作k8s的资源](https://github.com/helm/helm/issues/3130)

```bash
# 创建ServiceAccount
kubectl --namespace kube-system create serviceaccount tiller

# 给ServiceAccount设置权限
kubectl create clusterrolebinding tiller-cluster-rule \
 --clusterrole=cluster-admin --serviceaccount=kube-system:tiller

# 将上述账号与tiller关联
kubectl --namespace kube-system patch deploy tiller-deploy \
 -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'

```

当然也可以用一个`tiller.yaml`文件来一步[安装tiller](https://github.com/helm/helm/issues/3130#issuecomment-393744456)

```bash
kubectl create -f tiller.yaml
```

## Chart创建

`create`子命令用创建新的chart

```
helm create [chart-name]
```

其将会在当前目录下新建以`[chart-name]`命令的子目录, 其中包含helm所需要的项目结构及初始化文件.

```bash
# helm create nginx
$> tree nginx

nginx
├── Chart.yaml
├── charts
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── ingress.yaml
│   └── service.yaml
└── values.yaml

```

其中各种资源的模板存放`templates`下, 变量保存在`values.yaml`.

模板样例 - `templates/deployment.tpl`

{% raw %}
```yaml
# templates/deployment.tpl
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: {{ template "nginx.fullname" . }}
  labels:
    app: {{ template "nginx.name" . }}
    chart: {{ template "nginx.chart" . }}
    release: {{ .Release.Name }}
    heritage: {{ .Release.Service }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ template "nginx.name" . }}
      release: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ template "nginx.name" . }}
        release: {{ .Release.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
        # omit...
```
{% endraw %}

`templates`目录下每个文件都是golang [template文件](https://golang.org/pkg/text/template/). 其中双括号部分表示一个[Action](https://blog.gopheracademy.com/advent-2017/using-go-templates/), 其可能是数据求值(如`{{ .Values.repliaCount }}`), 或者函数调用(如`{{ template "nginx.name" . }}`), 或者是执行某个表达式等.

模板中的数据来源有两个: 一个是`helm`内置的一些变量, 如`.Release.Name`/`.Chart.Name`, 详细见官网[Built-in Objects](https://helm.sh/docs/chart_template_guide/builtin_objects/). 另一个是来自`values.yaml`, 以`.Values`开头.

values.yaml样例
```yaml
# values.yaml
# Default values for nginx.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: nginx
  tag: stable
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

# omit...
```

## Chart管理

- 部署chart

```bash
helm install --name [release-name] nginx
```
`release-name`用于指示当前部署的发布版本号, 可以看作是每次发布的ID. 同一个chart可以发布多次. 如果不指定,
helm将用内置的以及`values.yaml`中的变量去渲染`templates`下的所有文件, 并将更新后的`yaml`部署到K8s上.

在部署之前, 最好先检查chart是否有问题以及生成的yaml内容是否是自己期待的, helm分别提供了`lint`和`template`子命令.

```bash
# examines a chart for possible issues
helm lint [chart-name]

# render chart templates locally and display the output
# `release-name` is optional, default "RELEASE-NAME"
helm template --debug --name [release-name] [chart-name]
```

- 查看所有Release

helm将每次chart部署作为一个单独的release, 其管理单位也是release粒度.

```terminal
# list all releases
helm list --all
|> NAME	REVISION	STATUS  	CHART           NAMESPACE
|> test	1       	DEPLOYED	nginx-0.1.0	default
```

- 查看单个Release

```
helm list [release-name]
```

查看release对应的资源状态

```
helm status [release-name]
```

查看release生成的K8s资源YAML格式
```
helm get [release-name]
```

- 删除Release

```
helm delete [release-name] --purge
```

## 自定义模板

helm允许用户[自定义模板](https://helm.sh/docs/howto/charts_tips_and_tricks/), 也称为[named template/partial template/subtemplate](https://helm.sh/docs/chart_template_guide/named_templates/). 这些模板可以被其他模板使用, 以减少重复书写. 自定模板可以接受0个或者1个参数(也称为scope或者context??), 使用`include`函数来调用, 类似宏展开, 模板展开的内容将插入到调用处. (确切地说, golang的内置指令`template`也可以，但是其[不够灵活](https://v2.helm.sh/docs/chart_template_guide/#named-templates), 因此不推荐使用)

同时[helm内置了很多函数](https://v2.helm.sh/docs/charts_tips_and_tricks/#know-your-template-functions), 可以对数据进行处理. 结合这些函数, 自定义模板就可以很灵活.

比如, 在设置应用启动参数时, 经常需要以逗号分隔的列表值. 如果helm有相应的函数或者支持自定义函数, 那是最方便的, 但是目前这些不支持(??). 另一种可行的方法就是自定义模板, 同样可以达到目的, 具体实现如下(参考了[stackoverflow](https://stackoverflow.com/questions/47668793/helm-generate-comma-separated-list))

{% raw %}
```
{{/*
函数功能: 根据前缀与个数, 生成以逗号分隔的列表, 如[prefix]-0,[prefix]-1,...
参数为dict类型, 在模板内部以`.`表示.
*/}}
{{- define "commalist" -}}
{{- $p := .prefix -}}
{{- $n := .count -}}
{{- $m := dict "lst" (list) -}}
{{- range $index, $_ := (int $n | until) -}}
{{- $_ := printf "%s-%d" $p $index | append $m.lst | set $m "lst" -}}
{{- end -}}
{{- join "," $m.lst -}}
{{- end -}}

{{ /* 应用样例--将创建字符串, "zk-0,zk-1,zk-2" */ }}
{{- define "commalist-sample" -}}
{{- include "commalist" (dict "prefix" "zk" "count" "3") -}}
```
{% endraw %}

上面模板中用到了很多[sprig](https://masterminds.github.io/sprig/)库中定义的以及golang内置的函数-[append/list](https://masterminds.github.io/sprig/lists.html), [int](https://masterminds.github.io/sprig/conversion.html), [util](https://masterminds.github.io/sprig/integer_slice.html), [dict/set](https://masterminds.github.io/sprig/dicts.html), [join](https://masterminds.github.io/sprig/string_slice.html), [range/printf](https://golang.org/pkg/text/template/#hdr-Functions) (*builtin*).

此外`helm`两个特殊的记号
- `$`: 根变量(root scope), 对于一个chart, 它是固定的.
- `.`: 当前变量, 随着上下文的不同, 其值也相应改变.

## Tips

- [Jekyll escape curly brace](https://stackoverflow.com/questions/24102498/escaping-double-curly-braces-inside-a-markdown-code-block-in-jekyll)

```
{{ "{% raw " }}%}
Hello, my name is {{ "{{ name " }}}}.
{{ "{% endraw " }}%}
```

## Note
1. Helm v2, <https://v2.helm.sh/>
2. Helm v3, <https://helm.sh/>
3. Helm v3 Chart Development Tips and Tricks, <https://helm.sh/docs/howto/charts_tips_and_tricks/>
4. jekyll escape liquid template tags, <https://stackoverflow.com/questions/3426182/how-to-escape-liquid-template-tags>
