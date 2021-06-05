---
layout: post
title: prometheus与grafana使用
category: tech
guid: 122418DF-33AA-4ADF-A5E2-95BFA6CFF515
tags: [tech, grafana, monitoring]

---
{% include JB/setup %}

Prometheus用于收集及存储数据, 而grafana用于展示数据, 它们经常配合使用, 尤其是用于监控云部署的应用
.使用中关键的是配置Prometheus的收集规则及grafana的显示规则, 本文将对这两部分进行介绍.

## 1. Install

一般来说, Promethues/Grafana都会是一个基本的K8s组件, 在创建K8s集群时一般都会启动它们. 本文将以最方便--Docker-来启动这两个服务. (完整的K8s yaml文件见链接-TODO)

```bash
# start prometheus
docker run -it -p 9090:9090 prom/prometheus:v2.19.2

```
