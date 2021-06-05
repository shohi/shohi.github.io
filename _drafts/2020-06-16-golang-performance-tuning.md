---
layout: post
title: Golang性能分析
category: tech
guid: E385D2A6-5070-4D82-8C41-1B11C05CDFF7
tags: [tech, golang, performance]

---
{% include JB/setup %}

### Library
library性能分析, 可用以下方法
- [Profiling](https://golang.org/pkg/runtime/pprof/)
- [Escape Analysis](http://www.agardner.me/golang/garbage/collection/gc/escape/analysis/2015/10/18/go-escape-analysis.html)
- [SSA Analysis](https://draveness.me/golang/docs/part1-prerequisite/ch02-compile/golang-ir-ssa/)


#### 1. Profiling
利用`go test`启用相应的Profile, 然后用`go tool pprof`进行解析分析. 常用的Profile, 在`go test`中都有对应的flag, 详情可查看帮助文档 - `go help testflag`.

```bash
# test help
go help test

# test flags help
go help testflag

# run test with profiling on
go test -cpuprofile cpu.prof -memprofile mem.prof -bench .

# or benchmark
go test -gcflags '-m -m -l' \
    -benchmem \
    -benchtime 5s \
    -run=^$ \
    -bench=[pattern] \
    -memprofile mem.out \
    -cpuprofile cpu.out
```

当然, 也可以用编程的方式自定义要进行性能分析的代码([link](https://welcome112s.com/2020/06/19/yuque/Go%20profile%20%E8%B0%83%E4%BC%98/))

```go
package main

import (
  "runtime/pprof"
)

func main() {
    var w io.Writer

    // start cpu profiling
    pprof.StartCPUProfile(w io.Writer)

    // code goes here

    // stop cpu profiling
    pprof.StopCPUProfile()
}

```

#### 2. Escape Analysis

逃逸分析是compiler用以决定在何处分配程序创建的变量. 如果变量是否在其被声明函数的外部所引用,　那么
这些变量的生命周期长于其创建时的函数, 因此需要将这些变量分配到Heap上, 逃逸的变量会导致GC压力以及内存分配的低效([Go Escape Analysis Flaw](https://docs.google.com/document/d/1CxgUBPlx9iJzkz9JWkb6tIpTe5q32QDmz8l0BouG0Cw/preview))

> Anytime a value is shared outside the scope of a function’s stack frame,
> it will be placed (or allocated) on the heap.

`go`使用`gcflags`来打开Escape Analysis, 其中有两个常中的选项
- `-m` -- print verbose escape analysis information. pass the `-m` option multiple times, which makes the output more
- `-l` -- prevents functions from being inlined

`gcflags`适用于`go run/build/test`, 如

```bash
go test -run=- -bench=Perf -gcflags '-m -l'
```

- `go build` - 分析library
- `go run` - 分析main
- `go test` - 分析特定的test或benchmark


值得注意的是golang的Escape Analysis还有很多可以改进的地方, 一些不需要分配在Heap的情况compiler并没
有检查出来, 需要经常查看golang相关的issue以及多做测试, 常见的flaw有以下几种([link-1](https://www.ardanlabs.com/blog/2018/01/escape-analysis-flaws.html), [link-2](https://docs.google.com/document/d/1CxgUBPlx9iJzkz9JWkb6tIpTe5q32QDmz8l0BouG0Cw/edit#))

1. Indirect Assignment
2. Indirect Call (Anonymous Function)
3. Slice and Map Assignments
4. Interfaces

另外, 在Escape Analysis输出的结果中有时会显示`leaking param`, 这并不表示有内存泄漏而仅仅指
[the param is kept alive even after returning](http://www.golangbootcamp.com/book/tricks_and_tips).


### 3. SSA Analysis
SSA(Static Single Assignment)分析是使用SSA工具来导出golang程序编译后的中间码, 以揭示最终编译后的执行指令.([中间代码生成](https://draveness.me/golang/docs/part1-prerequisite/ch02-compile/golang-ir-ssa/))

```bash
GOSSAFUNC=[your-func] go build
```

`GOSSAFUNC`也可用于`go test`.

### 4. Assembly Analysis

可以通过`gcflags=-S`来直接输出汇编代码, `go run/test/build`都可以使用, 因为这些子命令最终
都会调用`go compile`.

```bash
# -l to disable inlining
go test -gcflags="-l -S"

```

### HTTP Application

#### Profiling Handlers

Golang默认启动了输出各种profiler的handlers, Web应用直接将这些handlers加到Server的route中
就能使用, `pprof`支持用URL制定profiles(见下文).

```go

func profileRoutes(router *mux.Router) {
	// profiling handlers
	pprofHandlers := map[string]http.HandlerFunc{
		"/debug/pprof/":             pprof.Index,
		"/debug/pprof/cmdline":      pprof.Cmdline,
		"/debug/pprof/profile":      pprof.Profile,
		"/debug/pprof/symbol":       pprof.Symbol,
		"/debug/pprof/trace":        pprof.Trace,
		"/debug/pprof/heap":         pprof.Handler("heap").ServeHTTP,
		"/debug/pprof/allocs":       pprof.Handler("allocs").ServeHTTP,
		"/debug/pprof/goroutine":    pprof.Handler("goroutine").ServeHTTP,
		"/debug/pprof/threadcreate": pprof.Handler("threadcreate").ServeHTTP,
		"/debug/pprof/block":        pprof.Handler("block").ServeHTTP,
	}

	for k, v := range pprofHandlers {
		router.Handle(k, v)
	}
}

```


### pprof

`go tool pprof`是查看各种profiles的工具, 通常有两种使用方式 -- 交互式终端或者Web页面, 启动命令为

```bash
go tool pprof [-http=:port] [binary] [/path/to/profile]

# example 1 - local files
go tool pprof [binary] ./cpu.profile

# example 2 - http url
go tool pprof "http://$SERVER:$PORT/debug/pprof/profile"
```

如果不指定`-http`参数, 那么就是交互式终端方式, 否则就是Web方式. 在Web方式中, 可以直接通过URL来查看
pprof分析的结果. 另外如前所述, `pprof`支持用URL指定profile.

在交互式界面中常用的命令有以下几个

```bash
# top functions consuming cpu or allocating memory
pprof>top [count]

# show details of given function
pprof>list [pattern]

# generate svg graph and open it in browse
pprof>web
```

##### 1. CPU

```shell
go tool pprof -http=:8080 "http://$SERVER:$PORT/debug/pprof/profile"
```

##### 2. Memory
```bash
# alloc space
go tool pprof -alloc_space mem.out

# alloc objects
go tool pprof -alloc_objects mem.out

# inuse space
go tool pprof -inuse_space mem.out

# inuse space
go tool pprof -inuse_objects mem.out

```

##### 3. Trace

```shell
curl -o trace.profile -X POST -d "seconds=3" "http://$SERVER:$PORT/debug/pprof/trace"

go tool trace trace.profile
```

##### 4. Block

##### 5. flamegraph

Go 1.11以后, flamegraph直接集成到了`go tool pprof` - 以Web形式查看profile即可看到flamegraph([link](https://github.com/uber-archive/go-torch)), 如

```bash
go tool pprof -http=":8081" [binary] [profile]
```

## Reference

1. 对Go程序进行可靠的性能测试, <https://talkgo.org/t/topic/102>
2. escape analysis issues, <https://github.com/golang/go/search?q=escape+analysis&type=Issues>
3. High Performance Go Workshop, <https://dave.cheney.net/high-performance-go-workshop/dotgo-paris.html#modern_processors_are_limited_by_memory_latency_not_memory_capacity>
4. go pprof性能分析, <https://wudaijun.com/2018/04/go-pprof/>

注: 文中所用到的是`go1.14.4 darwin/amd64`.
