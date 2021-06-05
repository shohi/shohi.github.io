---
layout: post
title: golang之module proxy
category: tech
guid: 2C217B97-FD58-45B5-B990-6045F912198E
tags: [tech]

---
{% include JB/setup %}

Go module在1.13之后日趋成熟, Go项目来越来越多的使用go.mod. 但是在实际工作中, 依赖有一部分是私有的并不能直接通过`go get`得到, 这就需要搭建一个module proxy, 将所有的以来请求导向这个proxy, 然后有proxy处理私有和公有库. 本文将介绍使用[athens](https://github.com/gomods/athens)搭建内部的module proxy.

## 1. 运行athens

```bash
# save module in current directory
docker run -d -v "$PWD:/var/lib/athens" \
   -e ATHENS_DISK_STORAGE_ROOT=/var/lib/athens \
   -e ATHENS_STORAGE_TYPE=disk \
   --name athens-proxy \
   --restart always \
   -p 3000:3000 \
   gomods/athens:latest

```

## 2. 运行goproxy

```bash
# use PWD as cache dir
docker run -it \
       -v $HOME/.ssh:/root/.ssh \
       -p 9090:8081 \
       -v $PWD:/go \
       goproxy/goproxy

```


## Questions

1. 如何拉取以branch指定的依赖?

2. 如何拉取以commit hash指定的依赖?
